---
title: "Spark + Elasticsearch构建推荐系统"
date: 2018-10-24T16:23:46+08:00
comments: true
categories: [
	"推荐系统",
	"Spark",
	"Elastic",
]
tags: [
	"Spark",
    "Elasticsearch",
    "Zeppelin"
]
---

Github [hb-chen/spark-elasticsearch-recommender](github.com/hb-chen/spark-elasticsearch-recommender)

- Zeppelin `0.8.0`
- Spark `2.3.2`
- Elasticsearch `6.3.2`

<!--more-->

#### 1.环境准备
> Mac OSX

##### Zeppeline
```bash
# http://www.apache.org/dyn/closer.cgi/zeppelin/zeppelin-0.8.0/zeppelin-0.8.0-bin-netinst.tgz
$ wget http://mirrors.shu.edu.cn/apache/zeppelin/zeppelin-0.8.0/zeppelin-0.8.0-bin-netinst.tgz
$ tar -zxf zeppelin-0.8.0-bin-netinst.tgz
$ cd zeppelin-0.8.0-bin-netinst

# 安装必要interpreter
$ ./bin/install-interpreter.sh --name md,elasticsearch
$ ./bin/zeppelin-daemon.sh start
```

##### Spark
```bash
# http://spark.apache.org/downloads.html
$ wget https://www.apache.org/dyn/closer.lua/spark/spark-2.3.2/spark-2.3.2-bin-hadoop2.7.tgz
$ tar -zxf spark-2.3.2-bin-hadoop2.7.tgz
```

##### Elasticsearch
```bash
# https://www.elastic.co/downloads/past-releases
# Elasticsearch + 6.3.2
$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.3.2.zip
$ unzip elasticsearch-6.3.2.zip

# ES-Hadoop + 6.3.2
$ wget https://artifacts.elastic.co/downloads/elasticsearch-hadoop/elasticsearch-hadoop-6.3.2.zip
$ unzip elasticsearch-hadoop-6.3.2.zip
```

###### Elasticsearch 矢量评分插件
- [muhleder/elasticsearch-vector-scoring](https://github.com/muhleder/elasticsearch-vector-scoring)

```bash
# 修改build.gradle，这样不必Checkout Elasticsearch 
# https://github.com/muhleder/elasticsearch-vector-scoring/issues/1#issuecomment-415267767
buildscript {
  repositories {
    jcenter()
    mavenLocal()
  }
  dependencies {
    classpath "org.elasticsearch.gradle:build-tools:6.3.2"
  }
}

apply plugin: 'idea'
apply plugin: 'java'
apply plugin: 'elasticsearch.esplugin'

licenseFile = rootProject.file('LICENSE')
noticeFile = rootProject.file('NOTICE')

esplugin {
  name 'elasticsearch-vector-scoring'
  description 'Provides a fast vector multiplication script.'
  classname 'com.gosololaw.elasticsearch.VectorScoringPlugin'
}

dependencies {
  compile "org.elasticsearch:elasticsearch:6.3.2"
}
```
```bash
# 插件安装
$ ./bin/elasticsearch-plugin install {file:///path/to/plugin.zip}
```

##### Python依赖库
```bash
$ pip install elasticsearch
$ pip install numpy
$ pip install tmdbsimple # 忽略，暂时未使用
```

##### [Movielens数据集](https://grouplens.org/datasets/movielens/)下载
```bash
$ cd data # 与zeppelin-0.8.0-bin-netinst同Path，note中配置PATH_TO_DATA = "../data/ml-latest-small"
$ wget http://files.grouplens.org/datasets/movielens/ml-latest-small.zip
$ unzip ml-latest-small.zip
```

#### 2.启动服务
##### Elasticsearch启动
```bash
$ ./bin/elasticsearch
```

##### Zeppelin配置及启动
```bash
$ cp conf/shiro.ini.template conf/shiro.ini
$ vim conf/shiro.ini
# 管理员账户密码
[users]
admin = 123456, admin

$ cp conf/zeppelin-env.sh.template conf/zeppelin-env.sh
$ vim conf/zeppelin-env.sh
# Spark配置
export SPARK_HOME=/{apache-spark-path}/spark-2.3.2-bin-hadoop2.7
export SPARK_SUBMIT_OPTIONS="--driver-memory 2G"

$ cp conf/zeppelin-site.xml.template conf/zeppelin-site.xml
$ vim conf/zeppelin-site.xml
# 根据需要可以修改zeppelin.server.port等配置

# 启动
$ ./bin/zeppelin-daemon.sh start
```

#### 3.Notebook
http://localhost:8080
```bash
# Create new interpreter
# md

# elasticsearch
elasticsearch.client.type http
elasticsearch.port	9200

# spark
# 添加Dependencies
artifact /{elasticsearch-hadoop-path}/elasticsearch-hadoop-6.3.2/dist/elasticsearch-spark-20_2.11-6.3.2.jar
```

#### 参考
- [使用 Apache Spark 和 Elasticsearch 构建一个推荐系统](https://github.com/IBM/elasticsearch-spark-recommender)