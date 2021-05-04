# 简介

Flink 是一个分布式的流处理框架，它能够对有界和无界的数据流进行高效的处理。**Flink 的核心是流处理**，当然它也能支持批处理，Flink 将批处理看成是流处理的一种特殊情况，即数据流是有明确界限的。这和 Spark Streaming 的思想是完全相反的，Spark Streaming 的核心是批处理，它将流处理看成是批处理的一种特殊情况， 即把数据流进行极小粒度的拆分，拆分为多个微批处理。

Spark Streaming 数据流的拆分：

![image-20210503194213190](/Users/lwl/Library/Application Support/typora-user-images/image-20210503194213190.png)

## 优点

- Flink 是基于事件驱动 (Event-driven) 的应用，能够同时支持流处理和批处理；
- 基于内存的计算，能够保证高吞吐和低延迟，具有优越的性能表现；
- 支持精确一次 (Exactly-once) 语意，能够完美地保证一致性和正确性；
- 分层 API ，能够满足各个层次的开发需求；
- 支持高可用配置，支持保存点机制，能够提供安全性和稳定性上的保证；
- 多样化的部署方式，支持本地，远端，云端等多种部署方案；
- 具有横向扩展架构，能够按照用户的需求进行动态扩容；
- 活跃度极高的社区和完善的生态圈的支持。

# 核心架构

Flink 采用分层的架构设计，从而保证各层在功能和职责上的清晰。如下图所示，由上而下分别是 API & 

Libraries 层、Runtime 核心层以及物理部署层：

## API & Libraries层

这一层主要提供了编程 API 和 顶层类库：

编程 API : 用于进行流处理的 DataStream API 和用于进行批处理的 DataSet API；

顶层类库：包括用于复杂事件处理的 CEP 库；用于结构化数据查询的 SQL & Table 库，以及基于批处理的机器学习库 FlinkML 和 图形处理库 Gelly。

##  Runtime 核心层

这一层是 Flink 分布式计算框架的核心实现层，包括作业转换，任务调度，资源分配，任务执行等功能，基于这一层的实现，可以在流式引擎下同时运行流处理程序和批处理程序。

## 物理部署层

Flink 的物理部署层，用于支持在不同平台上部署运行 Flink 应用。

## API

### SQL & Table API

SQL & Table API 同时适用于批处理和流处理，这意味着你可以对有界数据流和无界数据流以相同的语

义进行查询，并产生相同的结果。除了基本查询外， 它还支持自定义的标量函数，聚合函数以及表值函

数，可以满足多样化的查询需求。

###  DataStream & DataSet API

DataStream & DataSet API 是 Flink 数据处理的核心 API，支持使用 Java 语言或 Scala 语言进行调

用，提供了数据读取，数据转换和数据输出等一系列常用操作的封装。

### Stateful Stream Processing

Stateful Stream Processing 是最低级别的抽象，它通过 Process Function 函数内嵌到 DataStream 

API 中。 Process Function 是 Flink 提供的最底层 API，具有最大的灵活性，允许开发者对于时间和状

态进行细粒度的控制。

## 集群架构

Flink 核心架构的第二层是 Runtime 层， 该层采用标准的 Master - Slave 结构， 其中，Master 部分又包含了三个核心组件：Dispatcher、ResourceManager 和 JobManager，而 Slave 则主要是 TaskManager 进程。

## Data Source

### 内置Data Source

Flink Data Source 用于定义 Flink 程序的数据来源，Flink 官方提供了多种数据获取方法，用于帮助开

发者简单快速地构建输入流。

#### 基于文件构建

-  **readTextFile(path)**：按照 TextInputFormat 格式读取文本文件，并将其内容以字符串的形式返回。
- **readFile(fileInputFormat, path)** ：按照指定格式读取文件。
-  **readFile(inputFormat, filePath, watchType, interval, typeInformation)**：按照指定格式周期性的读取文件。

#### 基于集合构建

- **fromCollection(Collection)**：基于集合构建，集合中的所有元素必须是同一类型。
- **fromElements(T ...)**： 基于元素构建，所有元素必须是同一类型。
- **generateSequence(from, to)**：基于给定的序列区间进行构建。
- **fromCollection(Iterator, Class)**：基于迭代器进行构建。第一个参数用于定义迭代器，第二个参数用于定义输出元素的类型。
- **fromParallelCollection(SplittableIterator, Class)**：方法接收两个参数，第二个参数用于定义输出元素的类型，第一个参数 SplittableIterator 是迭代器的抽象基类，它用于将原始迭代器的值拆分到多个不相交的迭代器中。

#### 基于Socket构建

Flink 提供了 socketTextStream 方法用于构建基于 Socket 的数据流，socketTextStream 方法有以下四

个主要参数：

- **hostname**：主机名；
- **port**：端口号，设置为 0 时，表示端口号自动分配；
- **delimiter**：用于分隔每条记录的分隔符；
- **maxRetry**：当 Socket 临时关闭时，程序的最大重试间隔，单位为秒。设置为 0 时表示不进行重试；设置为负值则表示一直重试。

### 自定义Data Source

#### SourceFunction

除了内置的数据源外，用户还可以使用 addSource 方法来添加自定义的数据源。自定义的数据源必须

要实现 SourceFunction 接口

#### ParallelSourceFunction 和RichParallelSourceFunction

上面通过 SourceFunction 实现的数据源是不具有并行度的，即不支持在得到的 DataStream 上调用setParallelism(n) 方法。

如果你想要实现具有并行度的输入流，则需要实现 ParallelSourceFunction 或RichParallelSourceFunction 接口。

## Streaming Connectors

除了自定义数据源外， Flink 还内置了多种连接器，用于满足大多数的数据收集场景。

## Transformation

Flink 的 Transformations 操作主要用于将一个和多个 DataStream 按需转换成新的 DataStream。它

主要分为以下三类：

- **DataStream Transformations**：进行数据流相关转换操作；
- **Physical partitioning**：物理分区。Flink 提供的底层 API ，允许用户定义数据的分区规则；
- **Task chaining and resource groups**：任务链和资源组。允许用户进行任务链和资源组的细粒度的控制。

### Sink

在使用 Flink 进行数据处理时，数据经 Data Source 流入，然后通过系列 Transformations 的转化，最终可以通过 Sink 将计算结果进行输出，Flink Data Sinks 就是用于定义数据流最终的输出位置。Flink 提供了几个较为简单的 Sink API 用于日常的开发。

#### writeAsText

writeAsText 用于将计算结果以文本的方式并行地写入到指定文件夹下，除了路径参数是必选外，该方法还可以通过指定第二个参数来定义输出模式。

####  writeAsCsv

writeAsCsv 用于将计算结果以 CSV 的文件格式写出到指定目录，除了路径参数是必选外，该方法还支持传入输出模式，行分隔符，和字段分隔符三个额外的参数。

#### print \ printToErr

print \ printToErr 是测试当中最常用的方式，用于将计算结果以标准输出流或错误输出流的方式打印到控制台上。

####  writeUsingOutputFormat

采用自定义的输出格式将计算结果写出，上面介绍的 writeAsText 和 writeAsCsv 其底层调用的都是该方法。

#### writeToSocket

writeToSocket 用于将计算结果以指定的格式写出到 Socket 中。