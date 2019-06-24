---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Kafka
---

# 一. 简介
`kafkactl`是一个使用Golang开发的kafka命令行工具，用于管控kafka集群。例如 `查询topic列表`、`创建topic`、`消费topic`、`生成数据` 等等。  
项目源码在 [go-kafka tools](https://github.com/chenguolin/golang/tree/master/go-kafka/tools) 欢迎交流学习~

要求: kafka server版本 >= v0.10.0.x，某些命令需要更高版本（例如create topic需要v0.10.1.x）

1. [kafkactl for mac](https://github.com/chenguolin/golang/blob/master/go-kafka/tools/bin/mac/kafkactl)
2. [kafkactl for linux](https://github.com/chenguolin/golang/blob/master/go-kafka/tools/bin/linux/kafkactl)

`下载完成之后需要设置一下权限: chmod 755 kafkactl`

```
$ ./kafkactl help
kafkactl is a console util tool to access kafka cluster.

Usage:
  kafakctl <command> ...

Available Commands:
  help		  help about any command.
  list		  list topics, consumer groups.
  describe	describe topic, consumer group.
  create	  create topic, partition.
  delete	  delete topic, messages.
  consume	  consume from kafka topic.
  producer	write message 2 kafka topic.

Use "kafkactl <command> --help" for more information about a given command.
```

# kafkactl list
```
$ kafkactl list --help
Usage:
  kafakctl list <sub-command> ...

Available Sub Commands:
  topics            list all topics.
  consumer-groups   list all consumer groups.

Options:
  -brokers          kafka brokers address.

Examples:
  # List all topics.
  kafkactl list topics -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # List all consumer group.
  kafkactl list consumer-groups -brokers 127.0.0.1:9092,127.0.0.2:9092.

Use "kafkactl list --help" for more information about a given command.
```

## kafkactl list topics
$ ./kafkactl list topics -brokers 127.0.0.1:9092
```
1. __consumer_offsets
2. ethereum_block_transaction
3. kafka_topic_test
4. sky_rocket_user_wrong_answer
4. kafka_topic_test-20190624
5. TestAdmin_CreateTopic
6. TestClient_CreateTopic333
7. TestClient_CreateTopic
8. TestClient_CreateTopic33
9. TestClient_CreateTopic2
10. kafka_topic_test5
...
```

## kafkactl list consumer-groups
$ ./kafkactl list consumer-groups -brokers 127.0.0.1:9092
```
1. kafka_topic_test_group
```

# kafkactl describe
```
$ kafkactl describe --help
Usage:
  kafakctl describe <sub-command> ...

Available Sub Commands:
  topic		        describe a topic.
  consumer-group	describe a consumer group.

Options:
  -topic		  topic name.
  -group		  consumer group name.
  -brokers		kafka brokers address.

Examples:
  # Describe a topic.
  kafkactl describe topic -topic kafka_topic_test -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Describe a consumer group.
  kafkactl describe consumer-group -group consumer_example -brokers 127.0.0.1:9092,127.0.0.2:9092.

Use "kafkactl describe --help" for more information about a given command.
```

## kafkactl describe topic
$ ./kafkactl describe topic -topic kafka_topic_test -brokers 127.0.0.1:9092
```
partition-0: oldOffset=[224927], newOffset=[231090], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-1: oldOffset=[224612], newOffset=[230733], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-2: oldOffset=[224377], newOffset=[230530], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-3: oldOffset=[224484], newOffset=[230569], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-4: oldOffset=[224776], newOffset=[230825], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-5: oldOffset=[224339], newOffset=[230514], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-6: oldOffset=[225895], newOffset=[231984], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-7: oldOffset=[224735], newOffset=[230862], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-8: oldOffset=[225200], newOffset=[231321], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
partition-9: oldOffset=[225516], newOffset=[231634], replicas=[1], leader=[1], isr=[[1]], offlineReplicas=[[]]
```

## kafkactl describe consumer-group
$ ./kafkactl describe consumer-group -group kafka_topic_test_group -brokers 127.0.0.1:9092
```
State:  Stable
Protocol:  range
ProtocolType:  consumer
Members:
  logstash-0-e3299d1c-b82d-474c-a815-93c590c7e82e: ClientHost=[/192.168.130.111], ClientID=[logstash-0]
  logstash-0-92ccb082-480b-45d9-8962-55ff4ee5929b: ClientHost=[/192.168.130.111], ClientID=[logstash-0]
  logstash-0-552766df-1403-444a-b6d3-33aeff39be35: ClientHost=[/192.168.130.111], ClientID=[logstash-0]
  logstash-0-cccadc67-a85b-42bb-a5f1-5185a0e6082c: ClientHost=[/192.168.130.111], ClientID=[logstash-0]
  logstash-0-1cbf2b1c-9705-40de-8c7f-b989423ebe1a: ClientHost=[/192.168.130.111], ClientID=[logstash-0]
  logstash-0-62e51f07-8d39-4a7f-9c77-7d82587b2f51: ClientHost=[/192.168.130.111], ClientID=[logstash-0]
```

# kafkactl create
```
$ ./kafkactl create --help
Usage:
  kafakctl create <sub-command> ...

Available Sub Commands:
  topic        create new topic, require kafka server >= v0.10.1.0.
  partition    cincrease topic partition, require kafka server >= v1.0.0.0.

Options:
  -topic             topic name.
  -partition         topic partition count, default 1 partition.
  -replica           topic replica count, default 1 replica.
  -totalPartition    topic total partition count.
  -brokers           kafka brokers address.

Examples:
  # Create new topic, default 1 partition and 1 replica.
  kafkactl create topic -topic kafka_topic_test -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Create new topic, 3 partition and default 1 replica.
  kafkactl create topic -topic kafka_topic_test -partition 3 -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Create new topic, 3 partition and 2 replica.
  kafkactl create topic -topic kafka_topic_test -partition 3 -replica 2 -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Create new partition.
  kafkactl create partition -topic kafka_topic_test -totalPartition 6 -brokers 127.0.0.1:9092,127.0.0.2:9092.

Use "kafkactl create --help" for more information about a given command.
```

## kafkactl create topic
$ ./kafkactl create topic -topic kafka_topic_test -partition 3 -replica 1 -brokers 127.0.0.1:9092
  Successfully ~

## kafkactl create partition
$ ./kafkactl create partition -topic kafka_topic_test -totalPartition 6 -brokers 127.0.0.1:9092
  Successfully ~

# kafkactl delete
```
$ ./kafkactl delete --help
Usage:
  kafakctl create <sub-command> ...

Available Sub Commands:
  topic        delete a topic, require kafka server version >= v0.10.1.0.
  message      delete topic partition messages, require kafka server version >= v0.11.0.0.

Options:
  -topic        topic name.
  -partition    topic partition ID.
  -endOffset    topic partition end offset, whose offset is smaller than the given offset of the corresponding partition will be delete.
  -brokers      kafka brokers address.

Examples:
  # Delete a topic.
  kafkactl delete topic -topic kafka_topic_test -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Delete topic partition messages.
  kafkactl delete message -topic kafka_topic_test -partition 3 -endOffset 100000 -brokers 127.0.0.1:9092,127.0.0.2:9092.

Use "kafkactl delete --help" for more information about a given command.
```

## kafkactl delete topic
$ ./kafkactl delete topic -topic kafka_topic_test -brokers 127.0.0.1:9092
  Successfully ~

## kafkactl delete message
$ ./kafkactl delete message -topic kafka_topic_test -partition 0 -endOffset 2 -brokers 127.0.0.1:9092
  Successfully ~

# kafkactl consume
```
$ ./kafkactl consume --help
Usage:
  kafakctl consume ...

Available Options:
  -topic        consume topic name.
  -group        consumer group name, default generate random name.
  -partition    consume topic partition ID, default use partition 0.
  -start        start from newest or oldest, default newest read from lateset.
  -brokers      kafka brokers.

Examples:
  # Consume topics, random generate consumer group and default use partition 0 and consume from newest.
  kafkactl consume -topic kafka_topic_test -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Consume topics, default use partition 0 and consume from newest.
  kafkactl consume -topic kafka_topic_test -group kafka_topic_test_group -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Consume topics, default consume from newest.
  kafkactl consume -topic kafka_topic_test -group kafka_topic_test_group -partition 1 -brokers 127.0.0.1:9092,127.0.0.2:9092.

  # Consume topics.
  kafkactl consume -topic kafka_topic_test -group kafka_topic_test_group -partition 1 -start oldest -brokers 127.0.0.1:9092,127.0.0.2:9092.

Use "kafkactl consume --help" for more information about a given command.
```
  
$ ./kafkactl consume -topic kafka_topic_test -group kafka_topic_test_group -partition 1 -start newest -brokers 127.0.0.1:9092
```
{"offset":231805, "key":, "message":{"log":{"category":"access","@timestamp":"2019-06-24T18:39:05+0800","remote_addr":"120.12.32.147",,"request":"GET /unread/get.json?uid=1999999999 HTTP/1.0","status":"200","first_byte_commit_time":"7","request_time":"0.007","http_x_real_ip":"230.61.192.190","http_x_forwarded_for":"230.61.192.190","content_length":"-","sent_http_content_length":"-","body_bytes_sent":"33","http_cdn":"-"},"stream":"stdout","time":"2019-06-24T10:39:05.028588606Z","k8s":{"container_id":"7c42deb58da2","container_name":"test-write-feed-pre","host":"bjxd-k8sn-1-20","ns":"k8s-log","pod_ip":"120.22.210.122","labels":{"meta-track":"pre","meta-svc":"pre"}}}}
...
```

# kafkactl producer
```
$ ./kafkactl producer --help
Usage:
  kafakctl producer ...

Available Options:
  -topic	topic name.
  -key		message key.
  -message	message body.
  -brokers	kafka brokers.

Examples:
  # Write meesage 2 topics.
  kafkactl producer -topic kafka_topic_test -key key -message message -brokers 127.0.0.1:9092,127.0.0.2:9092.

Use "kafkactl producer --help" for more information about a given command.
```

$ ./kafkactl producer -topic kafka_topic_test -group -key key -message "write by chenguolin 20190624" -brokers 127.0.0.1:9092
  Successfully write 2 partition [0] offset [0] ~

