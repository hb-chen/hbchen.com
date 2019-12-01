---
title: "分布式系统限流服务-Golang&Redis"
date: 2019-05-05
draft: false
comments: true
categories: [
	"分布式",
]
tags: [
	"限流",
	"Redis",
	"Go",
]
---

本文参考两个使用Redis做限流的项目，分析分布式系统限流服务的实现，包括固定窗口以及滚动窗口的限流。

<!--more-->

**参考项目**

- Istio项目`mixer`模块的[adapter/redisquota](https://github.com/istio/istio/tree/master/mixer/adapter/redisquota)
- [go-redis/redis_rate](https://github.com/go-redis/redis_rate)

## 固定窗口
![fixed-window](/img/distributed/rate-window-fixed.png)

### go-redis/redis_rate
`redis_rate`使用**窗口标识**做`key`的字符串，用`INCRBY`统计窗口已使用流量`n`，且`key`在此窗口结束后自动过期。

**窗口标识**的生成：

- `slot` = `utime`(*时间戳`s`*)➗`udur`(*窗口大小`s`，`dur ≥ 1s`*)
- `key` = `name` + `slot`

> `utime`与系统时钟相关，考虑一种较为极端的情况，窗口大小`1s`，`A`、`B`两个系统时钟差`2s`，而`key`的过期时间与窗口大小相同`1s`，
假设`key`在窗口最后有更新则`key`的生存周期是`2s`，而恰好时钟相差`2s`，这样`A`、`B`系统的分布式限流就变成了单机限流。<br/>
虽然比较极端，但在使用时还是需要注意，根据需要可以适当加大`key`的过期时间，同时存活多个窗口，使存活窗口范围覆盖系统间的时钟误差。

```go
func (l *Limiter) AllowN(
	name string, maxn int64, dur time.Duration, n int64,
) (count int64, delay time.Duration, allow bool) {
	udur := int64(dur / time.Second)
	utime := time.Now().Unix()
	slot := utime / udur
	delay = time.Duration((slot+1)*udur-utime) * time.Second

	if l.Fallback != nil {
		allow = l.Fallback.Allow()
	}

	name = allowName(name, slot)
	count, err := l.incr(name, dur, n)
	if err == nil {
		allow = count <= maxn
	}

	return count, delay, allow
}
```

### adapter/redisquota/fixed_window

**redisquota**使用`lua`脚本实现，设计上使用一个带有`token`和`expire`的哈希表，`key`过期时间为`windowLength`，窗口在`key`过期或`timestamp`≥`expire`时重置。
另外`redisquota`根据场景做了**幂等**设计，同一`deduplicationid`在窗口有效期内不会重新分配`token`

> 因为`timestamp`与系统时钟相关，同样假设`A`、`B`两个系统时钟差`2s`(`A`>`B`)，窗口大小`10s`，如果窗口①由`A`系统时间戳触发，而窗口②由`B`系统时间戳触发(`A`在后`2s`没有流量)，
这样①实际窗口大小就是`12s`，这样继续到窗口③由`A`系统时间戳触发，②的实际窗口代销就是`8s`，所以在分布式环境下限流效果同样受系统时钟误差的影响。

```lua
local key_meta = KEYS[1]
-- local key_data = KEYS[2]

local credit = tonumber(ARGV[1])
local windowLength = tonumber(ARGV[2])
-- local bucketLength = tonumber(ARGV[3])
local bestEffort = tonumber(ARGV[4])
local token = tonumber(ARGV[5])
local timestamp = tonumber(ARGV[6])
local deduplicationid = ARGV[7]

-- lookup previous response for the deduplicationid and returns if it is still valid
--------------------------------------------------------------------------------
if (deduplicationid or '') ~= '' then
    local previous_token = tonumber(redis.call("HGET", deduplicationid .. "-" .. key_meta, "token"))
    local previous_expire = tonumber(redis.call("HGET", deduplicationid .. "-" .. key_meta, "expire"))

    if previous_token and previous_expire then
        if timestamp < previous_expire then
            return {previous_token, previous_expire - timestamp}
        end
    end
end

-- read or initialize meta information
--------------------------------------------------------------------------------
local info_token = tonumber(redis.call("HGET", key_meta, "token"))
local info_expire = tonumber(redis.call("HGET", key_meta, "expire"))

if (not info_token or not info_expire) or (timestamp >= info_expire) then
    info_token = 0
    info_expire = windowLength + timestamp

    redis.call("HMSET", key_meta, "token", info_token, "expire", windowLength + timestamp)
    -- set the expiration time for automatic cleanup
    redis.call("PEXPIRE", key_meta, windowLength / 1000000)
end

if info_token + token > credit then
    if bestEffort == 1 then
        local exceeded = info_token + token - credit

        if exceeded < token then
            -- return maximum available allocated token
            redis.call("HMSET", key_meta, "token", credit)

            -- save current request and set expiration time for auto cleanup
            if (deduplicationid or '') ~= '' then
                redis.call("HMSET", deduplicationid .. "-" .. key_meta, "token", token - exceeded, "expire", info_expire)
                redis.call("PEXPIRE", deduplicationid .. "-" .. key_meta, math.floor((info_expire - timestamp) / 1000000))
            end

            return {token - exceeded, info_expire - timestamp}
        else
            -- not enough available credit
            return {0, 0}
        end
    else
        -- not enough available credit
        return {0, 0}
    end
else
    -- allocated token
    redis.call("HMSET", key_meta, "token", info_token + token)

    -- save current request and set expiration time for auto cleanup
    if (deduplicationid or '') ~= '' then
        redis.call("HMSET", deduplicationid .. "-" .. key_meta, "token", token, "expire", info_expire)
        redis.call("PEXPIRE", deduplicationid .. "-" .. key_meta, math.floor((info_expire - timestamp) / 1000000))
    end

    return {token, info_expire - timestamp}
end
```

### Lua脚本获取时间戳

既然在分布式环境系统时钟有可能存在误差，那自然考虑将时间戳转到Redis中获取，消除时钟误差对窗口的影响，这里有一个限制条件是需要Lua脚本支持**随机写入**，
有关Redis Lua脚本**随机写入**请参考此文《[redis4.0之Lua脚本新姿势](https://yq.aliyun.com/articles/195914)》，结合这个特性可以将两个方案的时间戳由外部传入改为在Lua脚本中获取Redis系统时间，
我在[hb-go/pkg/rate](https://github.com/hb-go/pkg/tree/master/rate)包分别使用`SET`、`HSET`做了实现可以参考。

```lua
-- Redis version ≥ 3.2
redis.replicate_commands()
local now = redis.call('TIME')
timestamp = (tonumber(now[1]) * 1e6 + tonumber(now[2])) * 1e3
```

## 滚动窗口
![rolling-window](/img/distributed/rate-window-rolling.png)

滚动窗口在固定窗口基础上，将一个`window`拆成多个`bucket`，这样窗口根据`bucket`的大小向前移动。Redis的结构为哈希表：`token`、`bucket.token`、`bucket.timestamp`和`key`，
以及一个存储`bucket`历史数据的有序集合(*以`bucket`的时间戳排序*)

- `token`:窗口内已使用`token`数
- `bucket.token`:当前桶内已使用`token`数
- `bucket.timestamp`:当前桶的时间戳
- `key`:累计`bucket`数量

```lua
local key_meta = KEYS[1]
local key_data = KEYS[2]

local credit = tonumber(ARGV[1])
local windowLength = tonumber(ARGV[2])
local bucketLength = tonumber(ARGV[3])
local bestEffort = tonumber(ARGV[4])
local token = tonumber(ARGV[5])
local timestamp = tonumber(ARGV[6])
local deduplicationid = ARGV[7]

-- lookup previous response for the deduplicationid and returns if it is still valid
--------------------------------------------------------------------------------
if (deduplicationid or '') ~= '' then
    local previous_token = tonumber(redis.call("HGET", deduplicationid .. "-" .. key_meta, "token"))
    local previous_expire = tonumber(redis.call("HGET", deduplicationid .. "-" .. key_meta, "expire"))

    if previous_token and previous_expire then
        if timestamp < previous_expire then
            return {previous_token, previous_expire - timestamp}
        end
    end
end

-- read meta information
--------------------------------------------------------------------------------
local info_token = tonumber(redis.call("HGET", key_meta, "token"))
local info_bucket_token = tonumber(redis.call("HGET", key_meta, "bucket.token"))
local info_bucket_timestamp = tonumber(redis.call("HGET", key_meta, "bucket.timestamp"))

-- initialize meta
--------------------------------------------------------------------------------
if not info_token or not info_bucket_token or not info_bucket_timestamp then
    info_token = 0
    info_bucket_token = 0
    info_bucket_timestamp = timestamp

    redis.call("HMSET", key_meta,
        "token", info_token,
        "bucket.token", info_bucket_token,
        "bucket.timestamp", info_bucket_timestamp,
        "key", 0)
end


-- move buffer to bucket list if bucket timer is older than bucket window
--------------------------------------------------------------------------------
if (timestamp - info_bucket_timestamp + 1) > bucketLength then
    if tonumber(info_bucket_token) > 0 then
        local nextKey = redis.call("HINCRBY", key_meta, "key", 1)
        local value = tostring(nextKey) .. "." .. tostring(info_bucket_token)
        redis.call("ZADD", key_data, info_bucket_timestamp, value);
    end
    redis.call("HMSET", key_meta,
        "bucket.token", 0,
        "bucket.timestamp", timestamp)
end

local time_to_expire = timestamp - windowLength

-- reclaim tokens from expired records
--------------------------------------------------------------------------------
local reclaimed = 0
local expired = redis.call("ZRANGEBYSCORE", key_data, 0, time_to_expire)

for idx, value in ipairs(expired) do
    reclaimed = reclaimed + tonumber(string.sub(value, string.find(value, "%.")+1))
end

-- remove expired records
--------------------------------------------------------------------------------
redis.call("ZREMRANGEBYSCORE", key_data, 0, time_to_expire)

-- update consumed token
--------------------------------------------------------------------------------
if reclaimed > 0 then
    info_token = info_token - reclaimed;
    if info_token < 0 then
        info_token = 0
    end
    redis.call("HSET", key_meta, "token", info_token)
end

-- update the expiration time for automatic cleanup
--------------------------------------------------------------------------------
redis.call("PEXPIRE", key_meta, windowLength / 1000000)
redis.call("PEXPIRE", key_meta, windowLength / 1000000)

-- calculate available token
--------------------------------------------------------------------------------
local available_token = credit - info_token

-- check available token and requested token
--------------------------------------------------------------------------------

if available_token <= 0 then
    -- credit exhausted
    return {0, 0}
elseif available_token >= token then
    -- increase token and bucket.token by token
    redis.call("HINCRBY", key_meta, "token", token)
    redis.call("HINCRBY", key_meta, "bucket.token", token)

    -- save current request and set expiration time for auto cleanup
    if (deduplicationid or '') ~= '' then
        redis.call("HMSET", deduplicationid .. "-" .. key_meta, "token", token, "expire", timestamp + windowLength)
        redis.call("PEXPIRE", deduplicationid .. "-" .. key_meta, windowLength / 1000000)
    end

    return {token,  windowLength}
else

    if bestEffort == 0 then
        -- not enough token
        return {0, 0}
    end

    -- allocate available token only
    redis.call("HINCRBY", key_meta, "token", available_token)
    redis.call("HINCRBY", key_meta, "bucket.token", available_token)

    -- save current request and set expiration time for auto cleanup
    if (deduplicationid or '') ~= '' then
        redis.call("HMSET", deduplicationid .. "-" .. key_meta, "token", available_token, "expire", timestamp + windowLength)
        redis.call("PEXPIRE", deduplicationid .. "-" .. key_meta, windowLength / 1000000)
    end

    return {available_token, windowLength}
end
```

> 限流的窗口与Flink的[Window](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/operators/windows.html#tumbling-windows)类似，
对于固定窗口结合使用场景的需求，可以借鉴Flink的**Tumbling Window**接口设计提供`offset`
