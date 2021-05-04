# 简介

Apache Flume 是一个分布式，高可用的数据收集系统。它可以从不同的数据源收集数据，经过聚合后发送到存储系统中，通常用于日志数据的收集。Flume 分为 NG 和 OG (1.0 之前) 两个版本，NG 在 OG 的基础上进行了完全的重构，是目前使用最为广泛的版本。下面的介绍均以 NG 为基础。

# 架构

下图为 Flume 的基本架构图：

![image-20210504150455143](/Users/lwl/Library/Application Support/typora-user-images/image-20210504150455143.png)

## 基本架构

外部数据源以特定格式向 Flume 发送 events (事件)，当 source 接收到 events 时，它将其存储到一个或多个 channel ， channe 会一直保存 events 直到它被 sink 所消费。 sink 的主要功能从channel 中读取 events ，并将其存入外部存储系统或转发到下一个 source ，成功后再从 channel中移除 events 。

## 基本概念

### Event

Event 是 Flume NG 数据传输的基本单元。类似于 JMS 和消息系统中的消息。一个 Event 由标题和正文组成：前者是键/值映射，后者是任意字节数组。

### Source

数据收集组件，从外部数据源收集数据，并存储到 Channel 中。

### Channel

Channel 是源和接收器之间的管道，用于临时存储数据。可以是内存或持久化的文件系统：

- Memory Channel : 使用内存，优点是速度快，但数据可能会丢失 (如突然宕机)；
- File Channel : 使用持久化的文件系统，优点是能保证数据不丢失，但是速度慢。

### Sink

Sink 的主要功能从 Channel 中读取 Event ，并将其存入外部存储系统或将其转发到下一个Source ，成功后再从 Channel 中移除 Event 。

### Agent

是一个独立的 (JVM) 进程，包含 Source 、 Channel 、 Sink 等组件。

# 架构模式

## multi-agent flow

Flume 支持跨越多个 Agent 的数据传递，这要求前一个 Agent 的 Sink 和下一个 Agent 的 Source 都必须是 Avro 类型，Sink 指向 Source 所在主机名 (或 IP 地址) 和端口。

## Consolidation

日志收集中常常存在大量的客户端（比如分布式 web 服务），Flume 支持使用多个 Agent 分别收集日志，然后通过一个或者多个 Agent 聚合后再存储到文件系统中。

##  Multiplexing the flow

Flume 支持从一个 Source 向多个 Channel，也就是向多个 Sink 传递事件，这个操作称之为 Fan Out (扇出)。默认情况下 Fan Out 是向所有的 Channel 复制 Event ，即所有 Channel 收到的数据都是相同的。同时 Flume 也支持在 Source 上自定义一个复用选择器 (multiplexing selector) 来实现自定义的路由规则。











