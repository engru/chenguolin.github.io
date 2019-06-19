---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Kafka
---

1. 安装kafka
   执行命令：brew install kafka

2. 修改server.properties
    执行命令：vi /usr/local/etc/kafka/server.properties

    增加一行配置如下：listeners=PLAINTEXT://localhost:9092

3. 启动zookeeper
    执行命令： zkServer start

4. 以server.properties的配置，启动kafka
    kafka-server-start /usr/local/etc/kafka/server.properties

5. 新建session，查看kafka的topic
    kafka-topics --list --zookeeper localhost:2181

6. 启动kafka生产者
    kafka-console-producer --topic [topic-name]  --broker-listlocalhost:9092
    也支持从文件读取数据：kafka-console-producer --topic [topic-name]  --broker-listlocalhost:9092 < input

7. 启动kafka消费者
    * 新版Kafka：kafka-console-consumer --bootstrap-server localhost:9092 --topic [topic-name]
    * 老版Kafka：kafka-console-consumer --zookeeper localhost:9092 --topic [topic-name]

8. 创建新的Topic
     kafka-topics.sh --create --topic live_sdk_pull_stream -zookeeper 192.168.129.212:2181,192.168.129.213:2181,192.168.129.222:2181,192.168.129.223:2181 --partitions 3 --replication-factor 2

9. 列出所有的Topic
    kafka-topics --list --zookeeper localhost:2181
