---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Kafka
---

# 一. 简介
[kafka命令行工具kafkactl介绍](https://chenguolin.github.io/2017/08/07/Kafka-7-Kafka%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7kafkactl%E4%BB%8B%E7%BB%8D/) 上一篇文章介绍了如何使用 kafkactl 来使用Kafka，这篇文章会介绍如何使用 HTTP API接口方式来使用Kafka。

我把使用Kafka的场景，简单的分为如下
1. `运维排查`: 这种场景大部分是开发、运维用来排查问题使用，这种情况我们推荐使用 [kafkactl](https://github.com/chenguolin/go-kafka/tree/master/tools/bin)，只需要运行这个binary就可以，快速便捷。
2. `代码开发`: 这种场景是我们需要在业务代码里面去使用Kafka，这种情况下有2种使用方式
    + 引入 SDK: 业务直接使用 [kafka SDK](https://github.com/chenguolin/go-kafka)，对于golang项目引入非常方便，具体使用可以参考 [kafka use example](https://github.com/chenguolin/go-kafka/tree/master/example)，不好的地方在于需要多引入一个依赖。
    + HTTP API: 业务通过调用HTTP API接口方式使用 [kafka http](https://github.com/chenguolin/go-kafka-http)，这种方式业务不再需要依赖kafka sdk了，只需要通过常规的HTTP接口调用即可满足，但是有个不好地方在于HTTP API接口存在性能瓶颈，对于某些QPS很高场景不一定适合。

基于以上使用Kafka需求场景分析，我写了一个 kafka HTTP 服务，方便业务通过 HTTP 接口来使用Kafka。 

`这种需求在公有云场景下非常常见，公有云对外提供Kafka Paas服务，企业如果要对接Kafka服务，可以使用 SDK 也可以使用 HTTP API方式。`

# 二. 源码
1. 项目源码在 https://github.com/chenguolin/go-kafka-http
2. 代码内部会缓存 kafka client、consumer、producer 实例，避免每次 HTTP 请求处理的时候都去创建，节省资源占用

# 三. 使用
要求: `kafka server版本 >= v0.10.0.x，某些命令需要更高版本（例如create topic需要v0.10.1.x）`

HTTP API Service提供一下接口供业务使用  
1. 查询topic列表: `/v1/list/topics`
2. 查询consumer group列表: `/v1/list/consumer_groups`
3. 查看某个topic信息: `/v1/describe/topic`
4. 查看某个consumer group信息: `/v1/describe/consumer_group`
5. 创建一个新的topic: `/v1/create/topic`
6. topic创建新的partition: `/v1/create/partition`
7. 删除某个topic: `/v1/delete/topic`
8. 删除消息: `/v1/delete/message`
9. 消费topic: `/v1/consumer/messages`
10. 生产数据: `/v1/producer/messages`

## ① 查询topic列表
1. HTTP Method: `GET`
2. 请求参数

| 参数 | 类型 | 是否必须 | 备注 | 
|---|---|---|---|
| brokers | string | 是 | kafka broker地址，多个broker之间用 `,` 分割，例如 127.0.0.1:9092,127.0.0.2:9092 | 
3. 使用举例
   ```
   $ curl "http://localhost:38765/v1/list/topics?brokers=127.0.0.1:9092"

   {
      "meta": {
          "code": 0,
          "error": "",
          "request_id": "ac148a57-144f-478c-af8f-5536dc08f184",
          "request_uri": "/v1/list/topics?brokers=127.0.0.1:9092"
      },
      "response": {
          "topics": [
             "kafka_topic_test-20190624",
             "kafka_topic_test",
             "TestClient_CreateTopic33",
             "dummy",
             "kafka_cgl",
             ...
         ]
      }
   }
   ```

## ② 查询consumer group列表
1. HTTP Method: `GET`
2. 请求参数

| 参数 | 类型 | 是否必须 | 备注 | 
|---|---|---|---|
| brokers | string | 是 | kafka broker地址，多个broker之间用 `,` 分割，例如 127.0.0.1:9092,127.0.0.2:9092 | 
3. 使用举例
   ```
   $ curl "http://localhost:38765/v1/list/consumer_groups?brokers=127.0.0.1:9092"

   {
      "meta": {
          "code": 0,
          "error": "",
          "request_id": "1aae0f59-9a78-4cd8-8fe6-088d2e097c34",
          "request_uri": "/v1/list/consumer_groups?brokers=127.0.0.1:9092"
      },
      "response": {
          "groups": [
              "consumer_example",
              ...
          ]
      }
   }
   ```

## ③ 查看某个topic信息
1. HTTP Method: `GET`
2. 请求参数

| 参数 | 类型 | 是否必须 | 备注 | 
|---|---|---|---|
| brokers | string | 是 | kafka broker地址，多个broker之间用 `,` 分割，例如 127.0.0.1:9092,127.0.0.2:9092 | 
| topic | string | 是 | topic名称 | 
3. 使用举例
   ```
   $ curl "http://localhost:38765/v1/describe/topic?brokers=127.0.0.1:9092&topic=kafka_topic_test"

   {
      "meta": {
          "code": 0,
          "error": "",
          "request_id": "9b036f6a-1a8b-4c3c-9da5-851112e917c8",
          "request_uri": "/v1/describe/topic?brokers=127.0.0.1:9092&topic=kafka_topic_test"
      },
      "response": {
          "total_partition": 3,
          "partitions": [
              {
                  "id": 0,
                  "isr": [
                      0
                  ],
                  "leader": 0,
                  "new_offset": 243848,
                  "offline_replicas": null,
                  "old_offset": 230322,
                  "replicas": [
                      0
                  ]
              },
              ...
           ]
        }
   }
   ```

## ④ 查看某个consumer group信息
1. HTTP Method: `GET`
2. 请求参数

| 参数 | 类型 | 是否必须 | 备注 | 
|---|---|---|---|
| brokers | string | 是 | kafka broker地址，多个broker之间用 `,` 分割，例如 127.0.0.1:9092,127.0.0.2:9092 | 
| consumer_group | string | 是 | consumer group名称 |
3. 使用举例
   ```
   $ curl "http://localhost:38765/v1/describe/consumer_group?brokers=127.0.0.1:9092&consumer_group=consumer_example"

   {
      "meta": {
          "code": 0,
          "error": "",
          "request_id": "bedac8bf-6b75-49b7-a961-4a2072314648",
          "request_uri": "/v1/describe/consumer_group?brokers=127.0.0.1:9092&consumer_group=consumer_example"
      },
      "response": {
          "Err": 0,
          "GroupID": "consumer_example",
          "Members": {
              "sarama-0c26bf79-f570-4a63-9390-0c9188caf7d9": {
                  "ClientHost": "/127.0.0.1",
                  "ClientID": "sarama",
                  "MemberAssignment": "AAAAAAABABBrYWZrYV90b...",
                  "MemberMetadata": "AAEAAAABABBrYWZrYV90b3BpY190ZXN0..."
              }
          },
          "Protocol": "range",
          "ProtocolType": "consumer",
          "State": "Stable"
      }
   }
   ```

## ⑤ 创建一个新的topic
1. HTTP Method: `GET`
2. 请求参数

| 参数 | 类型 | 是否必须 | 备注 | 
|---|---|---|---|
| brokers | string | 是 | kafka broker地址，多个broker之间用 `,` 分割，例如 127.0.0.1:9092,127.0.0.2:9092 | 
| topic | string | 是 | topic名称 |
| partition_count | int | 否 | partition个数，默认为1，最大允许为100 |
| replica_count | int | 否 | 每个partition replica个数，默认为1，最大允许为10 |
| retention_time | int | 否 | 消息保留时长(单位秒)，默认为259200(即3天)，最大不超过2592000(即30天) |
3. 使用举例

## ⑥ topic创建新的partition

## ⑦ 删除某个topic

## ⑧ 删除消息

## ⑨ 消费topic

## ⑩ 生产数据

