---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Kafka
---

# 一. Kafka简介
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka_architecture.png?raw=true)

上面是Kafka的一个架构图，包括几个部分
1. Producer: 消息生产者，向Broker发送消息。
2. Consumer: 消息消费者，从Broker读取消息。
3. Broker: 处理节点，一个Kafka节点就是一个Broker，多个Broker可以组成一个Kafka集群。
4. Topic: 主题，Kafka根据Topic对消息进行归类，发布到Kafka集群的每条消息都需要指定一个Topic。
5. Partition: 分区，一个Topic可以分为多个Partition，每个Partition内部消息是有序的。
6. ConsumerGroup: 消费组，每个Consumer Group包括多个Consumer，一条消息可以发送到多个不同的Consumer Group，但是一个Consumer Group中只能有一个Consumer能够消费该消息。

`Producer发送消息`: 每个Topic包括一个或多个Partition，默认情况下Kafka根据消息的key来进行Partition的选取，即`hash(key) % numPartitions`，保证相同key的消息一定会发送到相同的Partition。当key为空的时候，随机找一个Partition发送消息。    
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka_produce.png?raw=true)

`Consumer消费消息`: Kafka采用拉取模型，由Consumer记录消费状态，每个Consumer互相独立并顺序读取每个Partition的消息。每个Topic允许被多个Consumer Group消费，每个Consumer Group包括多个Consumer实例，不同Consumer Group之间互相独立，消费相同数据。   
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka_consume.png?raw=true)

Producer生产的消息会保存在Kafka集群中，不管消息有没有被消费，用户可以通过设置`保留时长`来清理过期的数据。

# 二. Consumer Group
每个Topic可以对应1个或多个Consumer Group，用Consumer Group可以将Consumer进行自由的分组而不需要多次发送消息到不同的Topic。每个Consumer Group包含1个或多个Consumer实例，这些Consumer实例共同消费一个Topic，每个Partition只允许被Consumer Group中的一个Consumer实例消费。  
![](https://static001.infoq.cn/resource/image/68/22/68f2ca117290d8f438610923c108ce22.png)

Kafka `0.9`版本之前使用Zookeeper管理Consumer Group，`0.9`版本之后对于每个Consumer Group会选择一个Broker作为消费组的协调者(Coordinator)，Coordinator负责管理Consumer Group的状态，它的主要工作是负责Rebalance: 当有新Consumer实例加入、旧Consumer实例退出或者Topic的metadata发生变化的时候，负责重新协调Partition的分配。

Consumer Group与Partition的关系，有以下几个特点
1. 1个Partition只能被同个Consumer Group的一个Consumer实例消费，同组的Consumer实例之间起到负载均衡的作用。因此，为了提高并发量可以增加Partition的数量，但是Partition过多会导致Replica副本拷贝的网络请求增加，故障耗时增加，也会导致Broker和Zk的内存增加。
2. 当Consumer实例数量多于Partition数量时，多余的Consumer实例空闲，例如4个Partition则最多被同一个Consumer Group中4个Consumer实例消费。`最佳实践是: 每个Consumer Group的Consumer消费实例个数和Partition个数保持一致`。
3. 不同Consumer消费消息时不是严格有序的，Kafka只能保证同一个Partition上消息是有序的，多个Partition之间的消息无法保证。

我们可以通过以下实验验证
1. Consumer Group 1个Consumer实例，消费所有Partition
   ![](http://www.dengshenyu.com/assets/kafka-consumer/one.png)
2. Consumer Group 2个Consumer实例，每个实例消费2个Partition
   ![](http://www.dengshenyu.com/assets/kafka-consumer/two.png)
3. Consumer Group 4个Consumer实例，每个实例消费1个Partition
   ![](http://www.dengshenyu.com/assets/kafka-consumer/four.png)
4. Consumer Group 超过4个Consumer实例，有部分实例空闲
   ![](http://www.dengshenyu.com/assets/kafka-consumer/more.png)
5. 2个Consumer Group，每个Group消费重复的数据
   ![](http://www.dengshenyu.com/assets/kafka-consumer/double.png)

通过以下命令查看某个Consumer Group的Partition分配情况  
`kafka-consumer-groups.sh --new-consumer --describe --group consumer-tutorial-group --bootstrap-server localhost:9092`
```
GROUP, TOPIC, PARTITION, CURRENT OFFSET, LOG END OFFSET, LAG, OWNER
consumer-tutorial-group, consumer-tutorial, 0, 6667, 6667, 0, consumer-1_/127.0.0.1
consumer-tutorial-group, consumer-tutorial, 1, 6667, 6667, 0, consumer-2_/127.0.0.1
consumer-tutorial-group, consumer-tutorial, 2, 6666, 6666, 0, consumer-3_/127.0.0.1
```

# 三. Offset
Consumer Group第一次初始化时，Consumer实例通常会读取每个Partition的最早或最新的Offset，然后顺序地读取每个Partition的数据，在消费者读取过程中，它会提交已经成功处理的消息的Offset，默认情况Consumer实例会定期提交Offset。

![](https://images.weserv.nl/?url=http://img.blog.csdn.net/20160221172517706)

Kafka `0.9`版本之后把Offsert保存到了`__consumer_offsert`的Topic下，这个`__consumer_offsert`有`50`个Partition，通过`hash(Consumer Group) % 50`来确定要保存到哪个Partition。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kafka_consume_offset.png?raw=true)

# 四. Partition
虽然可以通过增加Partition个数来提高并发，但是Partition并不是越多越好。

1. `Partition越多Broker和Zk需要使用的内存就越多`  
   Kafka 0.8.2之后，Producer端有个参数`batch.size`默认是16KB，它会为每个Partition缓存消息，一旦满了就打包将消息批量发出。因为这个参数是Partition级别的，所以Partition数越多，这部分缓存所需的内存占用也会更多。同理，Zk内部在内存中也维护了Partition级别的缓存，因此Partition数越多，这种缓存的成本就越大。
2. `Partition越多文件句柄数就越多`  
   每个Partition在底层文件系统都有属于自己的一个目录，该目录下通常会有两个文件`base_offset.log`和`base_offset.index`。很明显，如果Partition数越多，所需要保持打开状态的文件句柄数也就越多，最终可能会超过你的ulimit -n的限制。
3. `Partition数越多故障恢复时间越长`  
   Kafka通过Replica机制来保证高可用，具体做法就是为每个Partition保存若干个副本数，每个副本保存在不同的Broker上。其中的一个副本充当Leader副本，负责处理Producer和Consumer请求，其他副本充当Follower角色，由Kafka controller负责保证与Leader的同步。如果Leader所在的Broker挂掉了，Contorller会检测到然后在Zookeeper的帮助下重选出新的Leader，这个过程会导致Partition不可用。如果每个Broker上有很多Partition，当这个Broker挂掉的时候，Zookeeper和Controller需要立即对这个Broker上的Partition进行Leader选举，故障恢复的时间会变长。

