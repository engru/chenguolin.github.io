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

# 三. 使用

