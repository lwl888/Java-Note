# 介绍

InfluxDB 是用Go语言编写的一个开源分布式时序、事件和指标数据库，无需外部依赖。

InfluxDB在DB-Engines的时序数据库类别里排名第一。

# 重要特性

- 极简架构：单机版的InfluxDB只需要安装一个binary，即可运行使用，完全没有任何的外部依赖。
- 极强的写入能力： 底层采用自研的TSM存储引擎，TSM也是基于LSM的思想，提供极强的写能力以及高压缩率。
- 高效查询：对Tags会进行索引，提供高效的检索。
- InfluxQL：提供SQL-Like的查询语言，极大的方便了使用，数据库在易用性上演进的终极目标都是提供Query Language。
- Continuous Queries: 通过CQ能够支持auto-rollup和pre-aggregation，对常见的查询操作可以通过CQ来预计算加速查询。

# 时序数据库的需求

- 数十亿个单独的数据点
- 高写入吞吐量
- 高读取吞吐量
- 大型删除（数据过期）
- 主要是插入/追加工作负载，很少更新

# 基本语法

[中文文档](https://jasper-zhang1.gitbooks.io/influxdb/content/Guide/querying_data.html)

# Java API

- [ ] 文档

# InfluxDB 系统架构

![preview](https://pic3.zhimg.com/v2-5df0a8734422a15a601d301f0b1d4b1a_r.jpg)

## DataBase

用户可以通过 create database xxx 来创建一个database。

## Retention Policy（RP）

数据保留策略， 可用来规定数据的的过期时间。

```sql
CREATE RETENTION POLICY "a_year" ON "food_data" DURATION 52w REPLICATION 1 SHARD DURATION 1h
```

上述语句可以创建一个保存周期为52周的RP。REPLICATION 1 表示副本数量为1。 ”SHARD DURATION”指定每个Shard Group对应多长时间。

一个数据库可以用多个RP ， 不同的表可以设置不同的RP。

## Shard Group 

 实现了数据分区，但是Shard Group只是一个逻辑概念，在它里面包含了大量Shard，Shard才是InfluxDB中真正存储数据以及提供读写服务的概念。

## Share

![image-20210502222652859](/Users/lwl/Library/Application Support/typora-user-images/image-20210502222652859.png)

Share 就是上面章节所介绍的TSM 引擎， 负责把数据写入文件。

具体的过程类似于LSM，数据来了先存到cache， 等cashe 大小到一定程度就会异步flush 到TSM文件。 同时WAL（**Write Ahead Log**） 用来预防数据丢失。

# TSM原理

## 重要概念：

1. **Measurement** : 类似于mysql 的表名。 （census）
2. **tag key：** 类似于mysql 加了索引的列名
3. **Field key** : 类似于mysql 没加索引的列名
4. **Point**：类似SQL中一行记录，而并不是一个点。

```sql
insert census,location=1,scientist=langstroth butterflies=12,honeybees=231435362189575692182 
//向census 插入一条数据，location,scientist 为 tag key ，butterflies和honeybees 为filed key ，tag 和filed 之间空格间隔。 1435362189575692182为时间戳， 可省略。
show series from census
```

## InfluxDB 核心概念 – Series

时序数据的时间线就是**一个数据源采集的一个指标随着时间的流逝而源源不断地吐出数据**，这样形成的一条数据线称之为时间线。

InfluxDB在时序数据模型设计方面提出了一个非常重要的概念：**SeriesKey。SeriesKey实际上就是measurement+datasource(tags)**。时序数据写入内存之后按照SeriesKey进行组织。

时序数据在内存中表示为一个Map：<Key, List<Timestamp|Value>>， 其中Key = seriesKey + fieldKey。这个Map执行flush操作形成TSM文件。

TSM文件最核心的由Series Data Section以及Series Index Section两个部分组成，其中前者表示存储时序数据的Block，而后者存储文件级别B+树索引Block，用于在文件中快速查询时间序列数据块。

## Series Data Block

Map中一个Key对应一系列时序数据，因此能想到的最简单的flush策略是将这一系列时序数据在内存中构建成一个Block并持久化到文件。然而，有可能一个Key对应的时序数据非常之多，导致一个Block非常之大，超过Block大小阈值，因此在实际实现中有可能会将同一个Key对应的时序数据构建成多个连续的Block。但是，在任何时候，同一个Block中只会存储同一种Key的数据。

另一个需要关注的点在于，Map会按照Key顺序排列并执行flush，这是构建索引的需求。

## Series Index Block

每个key 对应一个Index Block。

很多时候用户需要根据Key查询某段时间（比如最近一小时）的时序数据，如果没有索引，就会需要将整个TSM文件加载到内存中才能一个Data Block一个Data Block查找，这样一方面非常占用内存，另一方面查询效率非常之低。

为了在不占用太多内存的前提下提高查询效率，TSM文件引入了索引，其实TSM文件索引和HFile文件索引基本相同。

Series Index Block由Index Block Meta以及一系列Index Entry构成：

1. Index Block Meta最核心的字段是Key，表示这个索引Block内所有IndexEntry所索引的时序数据块都是该Key对应的时序数据。
2. Index Entry表示一个索引字段，指向对应的Series Data Block。指向的Data Block由Offset唯一确定，Offset表示该Data Block在文件中的偏移量，Size表示指向的Data Block大小。Min Time和Max Time表示指向的Data Block中时序数据集合的最小时间以及最大时间，用户在根据时间范围查找时可以根据这两个字段进行过滤。

# TSM引擎工作原理－时序数据读取

TSM在启动之后就会将TSM文件的索引部分加载到内存，数据部分因为太大并不会直接加载到内存。

用户查询可以分为三步：

1. 首先根据Key找到对应的SeriesIndex Block，因为Key是有序的，所以可以使用二分查找来具体实现。
2. 找到SeriesIndex Block之后再根据查找的时间范围，使用[MinTime, MaxTime]索引定位到可能的Series Data Block列表。
3. 将满足条件的Series Data Block加载到内存中解压进一步使用二分查找算法根据timestamp查找即可找到。

