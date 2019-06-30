---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Fluentd
---

# 一. 简介
`Prometheus`和`Fluentd`都是[CNCF (Cloud Native Computing Foundation)](https://www.cncf.io/)成员项目，因此推荐使用`Prometheus`来监控Fluentd

关于`Prometheus`的基础和使用可以看以下2偏文章

1. [【prometheus】prometheus基础介绍](https://chenguolin.github.io/2019/02/15/Prometheus-Prometheus%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/)
2. [【prometheus】prometheus 监控k8s并在grafana上展示](https://chenguolin.github.io/2019/02/16/Prometheus-Prometheus-%E7%9B%91%E6%8E%A7K8s%E5%B9%B6%E5%9C%A8Grafana%E4%B8%8A%E5%B1%95%E7%A4%BA/)

Fluentd官方目前已经开源了一个prometheus插件，[fluent-plugin-prometheus](https://github.com/fluent/fluent-plugin-prometheus)，需要注意的是由于Fluentd版本的不同，我们所安装的`fluent-plugin-prometheus`插件版本也是不同，由于我们Fluentd版本为`v0.12`所以我们安装`fluent-plugin-prometheus`版本选择`0.5.0`版本。

1. 安装: `gem install fluent-plugin-prometheus -v "~> 0.5.0"`
2. 介绍: fluent-plugin-prometheus包括以下6个插件
    * `in_prometheus`: 插件用于暴露监控metrics指标，提供HTTP接口供Prometheus服务采集
    * `in_prometheus_monitor`: 插件用于Fluentd Output带buffer插件监控
    * `in_prometheus_output_monitor`: 插件用于Fluentd Ouput插件监控，比in_prometheus_monitor插件提供更多metrics指标
    * `in_prometheus_tail_monitor`: 插件用于Fluentd in_tail插件监控
    * `filter_prometheus`: 插件用于records相关统计，例如可以用于统计总输入records数
    * `out_prometheus`: 插件用于records相关统计，例如可以用于统计总输出records数
    
# 二. 使用
## ① in_prometheus
`in_prometheus`这个插件用于暴露监控metrics指标，提供HTTP接口供Prometheus服务采集。

fluent.conf里面常见配置如下
```
# input plugin that is required to expose metrics by other prometheus
# plugins, such as the prometheus_monitor input below.
<source>
  @type prometheus
  bind 0.0.0.0
  port 24231
  metrics_path /metrics
</source>

1. bind: bind地址，默认为0.0.0.0表示本机所有IPV4地址
2. port: 监听端口，默认为24231
3. metrics_path: HTTP uri path，默认为/metrics
```

1. 当fluent.conf添加了in_prometheus这个插件配置后，我们就可以使用`http://ip:23231/metrics`拉取prometheus metrics数据

## ② in_prometheus_monitor
`in_prometheus_monitor`这个插件插件用于Fluentd Output带有buffer插件监控

fluent.conf里面常见配置如下
```
# input plugin that collects metrics from MonitorAgent and exposes them
# as prometheus metrics
<source>
  @type prometheus_monitor
  # update the metrics every 5 seconds
  interval 5
  <labels>
     key1 valu1
     key2 valu2
     ...
  </labels>
</source>

1. interval: 更新监控metrics指标的时间间隔，默认为5s
2. labels: 用户加的labels
```

这个插件自带以下3个metrics，类型为`gauge`，默认labels为`plugin_id`(插件id)、`plugin_category`(插件分类)、`type`(插件名)。

我们可以使用PromQL做查询在Grafana做展示，关于PromQL的使用可以看[prometheus查询](https://chenguolin.github.io/2019/02/15/Prometheus-Prometheus%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/#%E5%85%AD-prometheus%E6%9F%A5%E8%AF%A2)

1. `fluentd_status_buffer_queue_length`: output插件 buffer queue长度  
   如图所示，通过`sum(fluentd_status_buffer_queue_length)`可以展示buffer queue的长度趋势图 （prometheus 每15s拉取一次数据）
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_status_buffer_queue_length.png?raw=true)
2. `fluentd_status_buffer_total_bytes`: output插件 buffer 总的size，包括map+queue两部分  
   如图所示，通过`sum(fluentd_status_buffer_total_bytes)`可以展示buffer 总的size趋势图  （prometheus 每15s拉取一次数据）
    ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_status_buffer_total_bytes.png?raw=true)
3. `fluentd_status_retry_count`: 错误重试次数  
    如图所示，通过`sum(fluentd_status_retry_count)`可以展示重试次数趋势图  （prometheus 每15s拉取一次数据）
    ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_status_retry_count.png?raw=true)

## ③ in_prometheus_output_monitor
`in_prometheus_output_monitor`这个插件用于Fluentd Ouput插件监控，比in_prometheus_monitor插件提供更多metrics指标

fluent.conf里面常见配置如下
```
# input plugin that collects metrics from MonitorAgent and exposes them
# as prometheus metrics
<source>
  @type prometheus_output_monitor
  interval 5
  <labels>
     key1 valu1
     key2 valu2
     ...
  </labels>
</source>

1. interval: 更新监控metrics指标的时间间隔，默认为5s
2. labels: 用户加的labels
```

这个插件默认自带以下6个metrics，类型为gauge，默认labels为`plugin_id`(插件id)、`type`(插件名)

我们可以使用PromQL做查询在Grafana做展示，，关于PromQL的使用可以看[prometheus查询](https://chenguolin.github.io/2019/02/15/Prometheus-Prometheus%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/#%E5%85%AD-prometheus%E6%9F%A5%E8%AF%A2)
    
1. `fluentd_output_status_buffer_queue_length`:  output插件 buffer queue长度  
   如图所示，通过`sum(fluentd_output_status_buffer_queue_length)`可以展示buffer queue长度趋势图  （prometheus 每15s拉取一次数据）
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_buffer_queue_length.png?raw=true)
2. `fluentd_output_status_buffer_total_bytes`: output插件 buffer 总字节，包括map+queue  
   如图所示，通过`sum(fluentd_output_status_buffer_total_bytes)`可以展示buffer queue长度趋势图  （prometheus 每15s拉取一次数据）
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_buffer_total_bytes.png?raw=true)

3. `fluentd_output_status_retry_count`: ouput插件 错误重试次数  
   如图所示，通过`sum(fluentd_output_status_retry_count)`可以展示错误重试次数趋势图  （prometheus 每15s拉取一次数据）
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_retry_count.png?raw=true)

4. `fluentd_output_status_num_errors`: output插件，错误次数
    如图所示，通过`sum(fluentd_output_status_num_errors)`可以展示错误次数趋势图  （prometheus 每15s拉取一次数据）
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_num_errors.png?raw=true)
   
5. `fluentd_output_status_emit_count`: outupt插件，总emit条数
   如图所示，通过`sum(rate(fluentd_output_status_emit_count)[2m])`可以展示每秒emit条数趋势图  （prometheus 每15s拉取一次数据）
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_emit_count.png?raw=true)

6. `fluentd_output_status_retry_wait`: output插件，重试等待时间
   
## ④ in_prometheus_tail_monitor
`in_prometheus_tail_monitor`这个插件用于Fluentd in_tail插件监控 【不推荐使用这个插件，因为会导致pos_file offest错乱 详见https://github.com/fluent/fluent-plugin-prometheus/issues/49】

fluent.conf里面常见配置如下
```
<source>
  @type prometheus_tail_monitor
  interval 5
</source>

1. interval: 更新监控metrics指标的时间间隔，默认为5s
```

这个插件默认自带以下2个metrics，类型为gauge，默认labels为`plugin_id`(插件id)、`type`(插件名)

1. `fluentd_tail_file_position`: 当前文件读到的offset
2. `fluentd_tail_file_inode`: 当前文件的inode

## ⑤ filter_prometheus
`filter_prometheus`这个插件可以用于records相关统计，例如可以用于统计总输入records数

fluent.conf里面常见配置如下，支持配置多个metric
```
<filter message>
  @type prometheus
  <metric>
    name input_records
    type counter
    desc The total number of input records.
  </metric>
  <metric>
    name message_foo_counter
    type counter
    desc The total number of foo in message.
    key foo
  </metric>
</filter>

1. metric: prometheus metrics配置
   name: metrics名称
   type: metrics类型
   desc: metrics描述
   key: 可选字段 是否只统计某个record的字段
```

配置完成后，对应会有2个metrics
1. input_records
2. message_foo_counter

## ⑥ out_prometheus
`out_prometheus`这个插件插件用于records相关统计，例如可以用于统计总输出records数

fluent.conf里面常见配置如下，支持配置多个metric
```
<match message>
  @type copy
  <store>
    @type prometheus
    <metric>
      name output_records
      type counter
      desc The total number of output records.
    </metric>
    <metric>
      name message_foo_counter
      type counter
      desc The total number of foo in message.
      key foo
    </metric>
  </store>
  <store>
    @type stdout
  </store>
</match>

1. metric: prometheus metrics配置
   name: metrics名称
   type: metrics类型
   desc: metrics描述
   key: 可选字段 是否只统计某个record的字段
```

配置完成后，对应会有2个metrics
1. output_records
2. message_foo_counter
