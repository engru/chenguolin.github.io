---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Kafka
---

# 一. 简介
![](https://raw.githubusercontent.com/chenguolin/chenguolin.github.io/master/data/image/kafka_architecture.png)

上面是Kafka的一个架构图，包括几个部分
1. Producer: 消息生产者，向Broker发送消息。
2. Consumer: 消息消费者，从Broker读取消息。
3. Broker: 处理节点，一个Kafka节点就是一个Broker，多个Broker可以组成一个Kafka集群。
4. Topic: 主题，Kafka根据Topic对消息进行归类，发布到Kafka集群的每条消息都需要指定一个Topic。
5. Partition: 分区，一个Topic可以分为多个Partition，每个Partition内部消息是有序的。
6. ConsumerGroup: 消费组，每个Consumer Group包括多个Consumer，一条消息可以发送到多个不同的Consumer Group，但是一个Consumer Group中只能有一个Consumer能够消费该消息。

Kafka被用于两大类应用
1. 在应用间构建实时的数据流通道
2. 构建传输或处理数据流的实时流式应用

几个概念
1. Kafka以集群模式运行在1或多台服务器上
2. Kafka以topics的形式存储数据流
3. 每一个记录包含一个key、一个value和一个timestamp

Kafka有4个核心API
1. Producer API：用于应用程序将数据流发送到一个或多个Kafka topics
2. Consumer API：用于应用程序订阅一个或多个topics并处理被发送到这些topics中的数据
3. Streams API：允许应用程序作为流处理器，处理来自一个或多个topics的数据并将处理结果发送到一个或多个topics中，有效的将输入流转化为输出流
4. Connector API：用于构建和运行将Kafka topics和现有应用或数据系统连接的可重用的produers和consumers。例如，如链接到关系数据库的连接器可能会捕获某个表所有的变更

Kafka 客户端和服务端之间的通信是建立在简单的、高效的、语言无关的 TCP 协议上的。此协议带有版本且向后兼容。  
我们为 Kafka 提供了 Java 客户端，但是客户端可以使用多种语言。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka.png?raw=true)

# 二. Topics and Logs
对于每个 Topic Kafka会会维护一个如下所示的分区日志

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka-topic.png?raw=true)

1. 每个分区是一个有序的，以不可变的记录顺序追加 Message，分区中的每个记录都有一个连续的 ID，称为 Offset，唯一标识分区内的记录。
2. Kafka 集群可以配置记录保留时长，如果配置策略为两天那么在一条 Message 发布两天内，它都是可以被消费的，之后将被丢弃以腾出空间。
3. Kafka 的性能和数据量无关，所以存储长时间的数据并不会成为问题，但是和 Partition 数量有关系

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka-consumer.png?raw=true)

1. 实际上唯一需要保存的元数据是消费者的消费进度，即消费日志的偏移量 Offset。这个 Offset 是由 Consumer 控制的，通常消费者会在读取 Message 时以线性方式commit Offset，但是事实上，由于 Offset 由 Consumer 控制，因此它可以以任何顺序消费记录。例如一个 Consumer 可以通过重置 Offset 来处理过去的数据或者跳过部分数据。`每个 partition 内每条消息由 offset 唯一标识，每个消费组会标识自己消费到哪个 offset`
2. 每个topic有多个分区，主要影响同时允许多少个 Consumer 消费该topic，影响的是并行消费的能力。

# 三. Distribution
1. 每个Topic Partition Message 分布在集群中 Broker 服务器中，每个服务器处理一部分分区的数据和请求。
2. 每个 Partition 可以配置多副本，多副本由 一个 Leader节点 和 多个 Follower节点 组成。
3. Leader 处理该 Partition 所有的读写请求，Follower 复制 Leader 数据。如果 Leader 节点宕机，将会有一个 Follower 节点自动的转化为 Leader

# 四. Producers
Producer 负责决定将数据发送到 Topic 的哪个 Partition 上，这可以通过简单的循环方式来平衡负载，或则可以根据某些语义来决定分区（例如基于数据中一些关键字）。

Producer在初始化的时候可以设置 request.required.acks 参数，用于设置写消息机制

1. `request.required.acks=0`: 表示 Producer 不等待来自broker同步完成确认，就继续发送下一批消息。这个配置能够提供最低的延迟，但当服务器发生故障时某些数据会丢失，如 replica leader挂掉了但 Producer 并不知情，发出去的信息 broker 可能就收不到。这个机制可以提供最佳的写性能。
2. `request.required.acks=1`: 表示 Producer 在 replica leader 已成功收到的数据并得到确认后，才会发送下一批消息。一般情况下使用当前机制即可保证数据不丢，但是某些场景下例如 replica leader 还未同步给 replica flower 副本就挂掉了，这个时候数据就有可能丢失了。
3. `request.required.acks=-1`: 表示 Producer 在所有 replica follower 副本都确认接收到数据后，才会发送下一批消息。这个机制可以保证只要有一个 replica 副本存活，那么数据就不丢。这个机制写性能最差。

# 五. Consumers
1. Consumer 使用一个 Group name来标识自己的身份，每条被发送到一个 Topic 的消息都将被分发到属于同一个 group 的 Consumer 的一个实例中。  
2. Group name 相同的 Consumer 属于一个组，一个 Topic 的一条消息会被这个组中的一个 Consumer实例消费。
3. 如果所有的 Consumer 实例都是属于一个 Group 的，那么所有的消息将被均衡的分发给每个实例；如果所有的 Consumer 都属于不同的 Group，那么每条消息将被广播给所有的 Consumer。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka-consumer-group.png?raw=true)

1. 上图一个包含两个 Server 的 Kafka 集群，拥有四个分区（P0-P3），有两个 Consumer group
   + Group A: 有C1、C2两个Consumer
   + Group B: 有C3、C4、C5、C6四个Consumer
2. 更常见的是，Topic有多个Consumer group，每个 Consumer group 包含多个Consumer实例，为了可伸缩性和容错性。
3. Kafka 中消费的实现方式是`公平`的将 Partition 分配给 Consumer，每一个时刻 Partition 都拥有它唯一的消费者。
4. Kafka 只保证同一个 Partition 内记录的顺序，而不是同一个 Topic 的不同 Partition 间数据的顺序。  
   每个 Partition 顺序结合按 Key 分配的能力，能满足大多数程序的需求。  
5. 如果需要全局的顺序，可以使用只有一个 Partition 的Topic，这意味着每个 Consumer Group 只能有一个 Consumer实例，因为一个 Partition 同一时刻只能被一个Consumer实例消费

