---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Kafka
---

#### 1. 安装kafka  
1. brew install kafka

#### 2. 安装zookeeper  
2. brew install zookeeper

#### 3. 修改server.properties  
1. vi /usr/local/etc/kafka/server.properties  
2. 增加一行配置如下: listeners=PLAINTEXT://localhost:9092

#### 4. 启动zookeeper  
1. zkServer start

#### 5. 以server.properties的配置启动kafka  
1. kafka-server-start /usr/local/etc/kafka/server.properties

#### 6. 查看kafka的topic  
1. kafka-topics --list --zookeeper localhost:2181

#### 7. 启动kafka生产者  
1. kafka-console-producer --topic [topic-name]  --broker-listlocalhost:9092

#### 8. 启动kafka消费者  
1. 新版Kafka：kafka-console-consumer --bootstrap-server localhost:9092 --topic [topic-name]  
2. 老版Kafka：kafka-console-consumer --zookeeper localhost:9092 --topic [topic-name]

#### 9. 创建新的Topic  
1. kafka-topics.sh --create --topic live_sdk_pull_stream -zookeeper 192.168.129.212:2181,192.168.129.213:2181,192.168.129.222:2181,192.168.129.223:2181 --partitions 3 --replication-factor 2

#### 10. 列出所有的Topic  
1. kafka-topics --list --zookeeper localhost:2181

