---
title: 前端性能监控平台-存储与计算架构展望
date: "2019-03-28T10:00Z"
description: 本文首先介绍网易云音乐自研前端性能监控平台的架构现状和当前系统架构遇到的痛点。随后介绍了NTSDB存储引擎可以解决的问题，并进一步给出更符合业界标准的监控平台存储与计算架构。
---

### 前言
本文首先介绍网易云音乐自研前端性能监控平台的架构现状和当前遇到的问题。随后介绍了NTSDB存储引擎可以解决的问题，并进一步给出更符合业界标准的监控平台存储与计算架构。文中NTSDB与业界通用架构主要是根据网易数据科学中心时序数据库领域专家-<a href='http://hbasefly.com/author/libisthanksgmail-com/' target='_blank'>范欣欣</a>给出的建议整理而来。

### 当前架构
![image](https://p1.music.126.net/tBao5AUpLEFlGenlxk4y0A==/109951163959092198.png)
浏览器端SDK采集的性能数据会经过Nginx负载均衡到NodeJS服务器, NodeJS服务器对上报数据做合法校验后, 直接将原始数据转发至InfluxProxy,InfluxProxy根据配置将数据按表分片至底层的InfluxDB节点。

#### Influx Proxy的集群的优势
当前我们的存储与计算架构的实现主要是依赖于InfluxProxy、InfluxDB所构成的计算存储集群。InfluxProxy为饿了么开源的[组件](https://github.com/shell909090/influx-proxy),主要提供了以下的功能：

##### 1.按measurement(数据库表名)做分片。
Proxy节点中会维护DB节点与measurement的映射关系,根据该配置，可将同一个数据库的表存入不同的DB节点, 以达到横向扩展的目的。配置示意如下：

```javascript
    {
        db1: ['table1'],
        db2: ['table2'],
        db3: ['table3'],
        ...
    }
```

##### 2. 数据备份能力。
InfluxDB 提供replication参数设置副本数，但单机版的副本在同一主机上，无法做到高可用。若在proxy上将同一张表配置在多个DB节点，在数据存入时，Proxy会将数据写入多个DB节点，达到数据备份的目的，在读取时Proxy会选择其中一个DB节点取出数据，以实现influxDB节点的高可用并提高读取性能。配置示意如下：
```javascript
    {
        db1: ['table1', 'table2', 'table3'],
        db2: ['table1', 'table2', 'table3']
    }
```
##### 3. 写失败时，缓存重试能力。
当底层的DB节点挂掉时，Proxy节点会将数据先写入本地文件中，待DB节点恢复后，Proxy节点会将数据重新写入DB节点。
##### 4. 高危查询语句过滤能力。
若查询语句中不通过where duration 指定查询范围, influx会将符合该查询的全部索引加载至内存中，会产生极大的性能开销。Proxy会过滤类似的高危查询语句。

#### 当前架构存在的痛点

##### 1. Proxy节点当前无高可用
Proxy节点为整个存储与计算的入口, 若Proxy节点挂掉，性能监控的全部存储与计算服务就挂掉了。当然这个问题并非特别棘手，后续可以通过搭建多台Proxy, 由NodeJS端做负载均衡来解决该问题。

##### 2. Proxy是对读写请求做了一层代理，非master slave集群模式
Proxy节点维护了表与DB实例的映射关系，做了一层数据读写的代理。但像创建Petention Policy(数据保留过期策略)、创建Continue Query(持续查询)等无法做代理和同步，需要手动连接至DB节点进行管理。这样会存在什么问题呢？

当只有一两个DB节点时, 这样手动管理并没有太大问题。但是当DB实例个数继续扩大后, 手动管理分片A实例、分片B实例、分片A副本实例、分片B副本实例, 若后续还有数据迁移，则CQ配置的管理将是一个噩梦。

##### 3. 扩容和数据迁移成本高
由于Proxy是按照表进行数据的分片, 假设一开始只用到了Proxy提供的数据备份能力即按以下进行配置：
```javascript
    {
        db1: ['table1', 'table2', 'table3', 'table4'],
        db2: ['table1', 'table2', 'table3', 'table4']
    }
```
当接入的应用数逐步增加后, 一台db无法承载3张表的存储与计算开销。我们做扩容的工作,比如新申请2台机器，把table3、table4拆分到新的机器上。
```javascript
    {
        db1: ['table1', 'table2'],
        db2: ['table1', 'table2'],
        db3: ['table3', 'table4'],
        db4: ['table3', 'table4'],
    }
```
除了修改Proxy的配置，我们还需要将原本db1 db2 上的数据迁移到db3、db4上。并且把原本针对table3、table4的CQ配置也迁移到db3、db4。

##### 4. 仅支持measurement层面的分片, 无法按数据分片
假设随着应用的增多，我们已经忍着剧痛，把数据拆成了这种地步：
```javascript
    {
        db1: ['table1'],
        db2: ['table2'],
        db3: ['table3'],
        db4: ['table4'],
    }
```
之后我们又会遇到新的问题：table1撑满了。。现在我们除了升级db1的机器，再没有别的办法扩容了。

### NTSDB
NTSDB为网易数据科学中心基于Influx自研的时序数据库。基本架构示意如下：
![image](https://p1.music.126.net/249sryZCD3i2-6gdvlJtAQ==/109951163959223735.png)

集群拥有3台master节点, 负责接收数据读写请求、同步数据库管理配置如创建Petention Policy(数据保留过期策略)、创建Continue Query(持续查询), master节点非常轻量。实际的存储与计算任务由Shard Server进行。

#### Influx存储模型简介
![image](https://p1.music.126.net/aMzLd3vJJGPydkRsUWQFYw==/109951163959253224.png)
在一张Influx数据库之上可以创建任意多个Rentention Policy(数据保留策略), 一个RP可以[配置](https://docs.influxdata.com/influxdb/v1.7/query_language/database_management/#create-retention-policies-with-create-retention-policy)如下参数：
- DURATION: 数据过期时间, 过期后的数据自动删除。
- REPLICATION: 副本数。
- SHARD DURATION: shard group的持续时长, 持续时间结束后会形成新的ShardGroup.
- SHARD BUCKET: 每个Shard Group包含的Shard个数。Influx DB单机版未提供该参数(默认为1)、NTSDB提供该配置。

举例来说当配置 
```shell
$CREATE RETENTION POLICY "rp_only_week" ON "wapm" >
DURATION 7d >
REPLICATION 2 > 
SHARD DURATION 1d  >
SHARD BUCKET 3;
```

我们创建了一个名为```rp_only_week```的RP, 其数据最长保留7天，副本数为2个，ShardGroup的持续时间为1天, 每个ShardGroup含有3个Shard。

存入数据时是可以指定RP(未指定时有默认RP), 我们可以将上报的原始数据存入7天的RP内, 将聚合过后的数据，存入一年过期的RP内。数据的过期，不是将每一条记录的时间与duration做对比，而是判断一个ShardGroup内的最新的数据是否已经过期，如果最新一条记录都过期了，则整个ShardGroup内的数据做批量删除，效率非常高。

当有数据存入时，influx 会将数据的measurement + tagKey1 + tagValue1 + tagKey2 + tagValue2 +... 形成SeriesKey, 并将SeriesKey 做hash运算后存入某个shard内。同一个ShardGroup内 Shard个数越多，读写性能越高。可以将Shard理解为Influx实际做存储与计算引擎。

回头来看下NTSDB的架构：
![image](https://p1.music.126.net/249sryZCD3i2-6gdvlJtAQ==/109951163959223735.png)
一个数据库的数据可分为多个shard落到不同的ShardServer上，每个shard都有自己的副本，存到不同的主机上，以保证高可用。并且我们可以横向无限地增加ShardServer的个数，当一台ShardServer无法承担一个Shard的压力时，我们可以调整SHARD BUCKET的数量，让数据均摊到其他节点。而节点的保活、备份、Petention Policy(数据保留过期策略)、Continue Query(持续查询)的管理都可以只连接到一台master上进行管理，master节点会自动同步给其他节点。

可以说NTSDB完美地解决了我们当前存储架构的痛点。

### 标准架构
我们当前的架构可以简单抽象为如下流程：
![image](https://p1.music.126.net/FDMpSUh5Cb-oPSOydP5fbQ==/109951163959325299.png)
我们将所有的原始数据存入Influx, 并在Influx上建立CQ做预聚合计算，以提高查询性能。

在此架构之上，我们计划再进一步，引进业界更为通用的存储计算架构：
![image](https://p1.music.126.net/SsF26FVK5sqc8xhWz1a4fw==/109951163959336654.png)

我们引入Kafka来做消息队列，有如下几点好处：
- 原本Server直接对Influx, Influx若挂掉，则这段时间内的数据就会完全丢失。引入消息队列后，数据可以先入队，后消费。
- 可应对高峰流量，高峰流量不常有，如果为了高峰流量而一直预备着高配机器，多少会是一种浪费，而引入kafka，influx就不需要有完全匹配高峰流量的配置，高峰时可在kafka先缓存，待高峰过后，逐步消费。

引入Flink做聚合计算，有如下几点好处：
- 更专业, 大数据分布式计算平台，提供更多的聚合函数，通过写SQL就可以完成聚合任务配置。
- 更灵活, 未提供的聚合函数，可通过开发JAR包的方式，灵活自定义配置。

网易云音乐基于Flink自研的[Magina平台](https://music-rtfm.hz.netease.com/magina-doc/)可简化Flink的使用，让大数据计算更加亲民。

文末再次感谢网易数据科学中心-时序数据库领域专家-<a href='http://hbasefly.com/author/libisthanksgmail-com/' target='_blank'>范欣欣</a>，对云音乐前端性能监控平台的架构改进提出的宝贵建议。

### 参考
- [Influx Db文档](https://docs.influxdata.com/influxdb/v1.7/query_language/database_management/#create-retention-policies-with-create-retention-policy)
- [刘平：饿了么Influxdb 实战解析](https://gitbook.cn/books/59428f6f7e850f039399fd02/index.html)
- [范欣欣-时序数据库技术体系 - 初识InfluxDB](http://hbasefly.com/2017/12/08/influxdb-1/)
- [范欣欣-时序数据库技术体系 - InfluxDB TSM存储引擎之TSMFile](http://hbasefly.com/2018/01/13/timeseries-database-4/)
- [范欣欣-时序数据库技术体系 - InfluxDB 多维查询之倒排索引](http://hbasefly.com/2018/02/09/timeseries-database-5/)
- [范欣欣-网易时序数据库，丰富你的技术栈](http://kms.netease.com/#/article/5933)