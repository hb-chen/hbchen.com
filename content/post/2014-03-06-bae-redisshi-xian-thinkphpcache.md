---
title: "BAE Cache&Redis实现ThinkPHP的Cache驱动"
date: 2014-03-06T19:00:00+08:00
comments: true
categories: [
    "PHP"
]
tags: [
    "ThinkPHP",
    "BAE",
    "Cache",
    "Redis",
]
description: [BAE Redis实现ThinkPHP的Cache驱动，以及BAE Cache驱动]
---

在BAE环境下有单独的Cache，同时也有Redis，可以同时用来做不同的Cache服务，先从Redis开始

1、Redis相关配置
```php
//conf.php
#BAE API Key与Secret Key
'BAE_AK' 	=> 'XXX',
'BAE_SK'	=> 'XXX',

#BAE Redis扩展配置
'BAE_REDIS_HOST'   =>	'redis.duapp.com',
'BAE_REDIS_PORT'   =>	80,
'BAE_REDIS_DBNAME' =>	'XXX',

```

<!--more--> 

可以选择将Redis是否设为默认缓存，如果不是，使用时注意切换
```
$cache = Cache::getInstance('Baeredis',array());
```

2、Redis Cache 驱动

```php
//CacheBaeredis.class.php
#根据CacheRedis.class.php修改

<?php

defined('THINK_PATH') or exit();


class CacheBaeredis extends Cache {
	 /**
	 * 架构函数
     * @param array $options 缓存参数
     * @access public
     */
    public function __construct($options=array()) {
        if ( !extension_loaded('redis') ) {
            throw_exception(L('_NOT_SUPPERT_').':redis');
        }
        if(empty($options)) {
            $options = array (
                'host'          => C('BAE_REDIS_HOST') ? C('BAE_REDIS_HOST') : '127.0.0.1',
                'port'          => C('BAE_REDIS_PORT') ? C('BAE_REDIS_PORT') : 80,
                'timeout'       => C('DATA_CACHE_TIMEOUT') ? C('DATA_CACHE_TIMEOUT') : false,
                'persistent'    => false,
            );
        }
        $this->options =  $options;
        $this->options['expire'] =  isset($options['expire'])?  $options['expire']  :   C('DATA_CACHE_TIME');
        $this->options['prefix'] =  isset($options['prefix'])?  $options['prefix']  :   C('DATA_CACHE_PREFIX');        
        $this->options['length'] =  isset($options['length'])?  $options['length']  :   0;        
            
        try {
            /*建立连接后，在进行集合操作前，需要先进行auth验证*/
            $this->handler = new Redis();
            $ret;
            if ($options['timeout'] === false) {
                $ret = $this->handler->connect($options['host'], $options['port']);
            }
            else {
                $ret = $this->handler->connect($options['host'], $options['port'], $options['timeout']);
            }

            if ($ret === false) {
                throw new RedisException($this->handler->getLastError());
            }

            $user = C('BAE_AK');
            $pwd = C('BAE_SK');
            $dbname = C('BAE_REDIS_DBNAME');

            $ret = $this->handler->auth($user . "-" . $pwd . "-" . $dbname);
            if ($ret === false) {
                throw new RedisException($this->handler->getLastError());
            }
         
        } catch (RedisException $e) {
            throw_exception('BAE Redis:'.$e->getMessage());
        }


    }

    /**
     * 读取缓存
     * @access public
     * @param string $name 缓存变量名
     * @return mixed
     */
    public function get($name) {
        N('cache_read',1);
        $value = $this->handler->get($this->options['prefix'].$name);
        $jsonData  = json_decode( $value, true );
        return ($jsonData === NULL) ? $value : $jsonData;	//检测是否为JSON数据 true 返回JSON解析数组, false返回源数据
    }

    /**
     * 写入缓存
     * @access public
     * @param string $name 缓存变量名
     * @param mixed $value  存储数据
     * @param integer $expire  有效时间（秒）
     * @return boolen
     */
    public function set($name, $value, $expire = null) {
        N('cache_write',1);
        if(is_null($expire)) {
            $expire  =  $this->options['expire'];
        }
        $name   =   $this->options['prefix'].$name;
        //对数组/对象数据进行缓存处理，保证数据完整性
        $value  =  (is_object($value) || is_array($value)) ? json_encode($value) : $value;

        //相对CacheRedis的驱动增加了expire>0的判断
        if(is_int($expire) & $expire > 0) {
            $result = $this->handler->setex($name, $expire, $value);
        }else{
            $result = $this->handler->set($name, $value);
        }
        if($result && $this->options['length']>0) {
            // 记录缓存队列
            $this->queue($name);
        }
        return $result;
    }

    /**
     * 删除缓存
     * @access public
     * @param string $name 缓存变量名
     * @return boolen
     */
    public function rm($name) {
        return $this->handler->delete($this->options['prefix'].$name);
    }

    /**
     * 清除缓存
     * @access public
     * @return boolen
     */
    public function clear() {
        return $this->handler->flushDB();
    }

}
```

3、BAE Cache驱动及配置

```php
#配置
#BAE API Key与Secret Key，前面已经有配置
'BAE_AK' 	=> 'XXX',
'BAE_SK'	=> 'XXX',

#设置自己的CacheID（资源名称）、Host和Port
'DATA_CACHE_TYPE' 	=> 'Bae',		//设为默认
'DATA_CACHE_ID'		=>	'XXX',
'MEMCACHE_HOST'		=>	'cache.duapp.com',
'MEMCACHE_PORT'		=>	000,


#require_once(BAE_API_ROOT_PATH . 'BaeMemcache.class.php');
#需要BAE相关的驱动文件，可以在index.php入口中添加Root Path方便使用，也可以自己修改定义
define('BAE_API_ROOT_PATH', '你的BAE驱动文件路径');

```

```php
//CacheBae.class.php
<?php
class CacheBae extends Cache {

    static $_cache;
    private $_handler;
   
    /**
     +----------------------------------------------------------
     * 架构函数
     +----------------------------------------------------------
     * @access public
     +----------------------------------------------------------
     */
    public function __construct($options='') {
        if(!empty($options)) {
            $this->options =  $options;
        }
        $this->options['expire'] = isset($options['expire'])?$options['expire']:C('DATA_CACHE_TIME');
        $this->options['length']  =  isset($options['length'])?$options['length']:0;
        $this->options['queque']  =  'bae';
        $this->init();
    }

    /**
     +----------------------------------------------------------
     * 初始化检查
     +----------------------------------------------------------
     * @access private
     +----------------------------------------------------------
     * @return boolen
     +----------------------------------------------------------
     */
    private function init() {
    	require_once(BAE_API_ROOT_PATH . 'BaeMemcache.class.php');
    	/*Cache配置信息，可查询Cache详情页*/
    	$cacheid = C('DATA_CACHE_ID');
    	$host = C('MEMCACHE_HOST');
    	$port = C('MEMCACHE_PORT');
    	$user = C('BAE_AK');
    	$pwd = C('BAE_SK');

		$this->_handler = new BaeMemcache($cacheid,$host. ': '. $port, $user, $pwd);
		$this->connected = true;
    }

    /**
     +----------------------------------------------------------
     * 是否连接
     +----------------------------------------------------------
     * @access public
     +----------------------------------------------------------
     * @return boolen
     +----------------------------------------------------------
     */
    private function isConnected() {
        return $this->connected;
    }
    /**
     +----------------------------------------------------------
     * 读取缓存
     +----------------------------------------------------------
     * @access public
     +----------------------------------------------------------
     * @param string $name 缓存变量名
     +----------------------------------------------------------
     * @return mixed
     +----------------------------------------------------------
     */
    public function get($name) {
        N('cache_read',1);
	$content = $this->_handler->get($name);
	if(false !== $content ){
            if(C('DATA_CACHE_COMPRESS') && function_exists('gzcompress')) {
		$content = substr($content,0,-1);  //remvoe \0 in the end
	    }
            if(C('DATA_CACHE_CHECK')) {//开启数据校验
                $check  =  substr($content,0, 32);
                $content   =  substr($content,32);
                if($check != md5($content)) {//校验错误
                    return false;
                }
            }
            if(C('DATA_CACHE_COMPRESS') && function_exists('gzcompress')) {
                //启用数据压缩
                $content   =   gzuncompress($content);
            }
            $content    =   unserialize($content);
	    return $content;
        }
        else {
            return false;
        }
    }

    /**
     +----------------------------------------------------------
     * 写入缓存
     +----------------------------------------------------------
     * @access public
     +----------------------------------------------------------
     * @param string $name 缓存变量名
     * @param mixed $value  存储数据
     * @param int $expire  有效时间 0为永久
     +----------------------------------------------------------
     * @return boolen
     +----------------------------------------------------------
     */
    public function set($name,$value,$expire=null) {
        N('cache_write',1);
        if(is_null($expire)) {
            $expire =  $this->options['expire'];
        }
        $data   =   serialize($value);
        if( C('DATA_CACHE_COMPRESS') && function_exists('gzcompress')) {
            //数据压缩
        //    $data   =   gzcompress($data,3);
	      $data =  gzencode($data) . "\0";
        }
        if(C('DATA_CACHE_CHECK')) {//开启数据校验
            $check  =  md5($data);
        }else {
            $check  =  '';
        }
	$data = $check.$data;
	$result =  $this->_handler->set($name,$data,0,intval($expire));
        if($result) {
            if($this->options['length']>0) {
                // 记录缓存队列
                $this->queue($name);
            }
	    return true;
        }else {
            return false;
        }
    }

    /**
     +----------------------------------------------------------
     * 删除缓存
     +----------------------------------------------------------
     * @access public
     +----------------------------------------------------------
     * @param string $name 缓存变量名
     +----------------------------------------------------------
     * @return boolen
     +----------------------------------------------------------
     */
    public function rm($name) {
        return $this->_handler->delete($name);
    }
    static function queueSet($name,$value)
    {
	$h = new BaeMemcache();
	if ( $h->set($name,$value) ){
		self::$_cache = array($name => $value);
	}
    }
    static function queueGet($name)
    {
	if(isset(self::$_cache[$name]))
		return self::$_cache[$name];
	$h = new BaeMemcache();
	$r = $h->get($name);
	if ( false === $r ){
		return false;
	}
	self::$_cache[$name] = $r;
	return $r;
    }

}

```
