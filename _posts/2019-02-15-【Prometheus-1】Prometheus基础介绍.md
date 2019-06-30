---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
---

# 一. Prometheus介绍
[Prometheus](https://prometheus.io/)是SoundCloud开源的监控告警系统，使用Golang开发。2012年开始编码，2015年在Github上开源，2016年加入[CNCF](https://cncf.io/)成为继K8s之后第二名成员。

`Prometheus`已经成为了云原生中指标监控的事实标准，几乎所有`Kubernetes`的核心组件以及其它云原生系统都以`Prometheus`的指标格式输出自己的运行时监控信息。

`Prometheus`目前最新版本为2.7，可以从[DOWNLOAD Prometheus](https://prometheus.io/download/)这个链接查看所有版本信息。

`Prometheus`主要功能如下
1. 多维数据模型，时序数据由Metric和多个Label组成。
2. PromQL灵活的查询语法。
3. 无依赖存储，支持本地和远程存储。
4. 通过HTTP协议采用PULL的方式拉取数据。
5. 可以采用服务发现或者静态配置方式，来发现目标服务。
6. 支持多种统计数据模型，UI优化，Grafana也支持Prometheus。

# 二. Prometheus架构
`Prometheus`官方架构如下图所示，Prometheus大部分组件使用`Go`开发，少部分组件使用Java、Python以及Ruby

![](https://prometheus.io/assets/architecture.svg)

从上面这个架构图可以看出，`Prometheus`的核心组件有以下6个
1. `Prometheus Server`: 负责采集和存储监控数据，并且对外提供PromQL实现监控数据的查询、聚合分析，以及告警规则管理。
    * `Retrieval`: 负责定时去暴露的目标服务上去拉取监控Metrics指标数据
    * `TSDB`: Prometheus 2.0版本起使用的底层存储, 它的数据块编码使用了facebook的gorilla，具备了完整的持久化方案
    * `HTTP Server`: 提供HTTP API接口查询监控Metrics数据
2. `Service discovery`: `Prometheus`支持多种服务发现机制，文件、DNS、Consul、Kubernetes、OpenStack、EC2等等。前提是这些服务需要开放`/metrics`接口提供数据查询。
3. `Prometheus targets`: `Prometheus`监控目标服务，`Prometheus`通过轮询这些目标服务拉取监控Metrics数据。
4. `Push gateway`: 某些场景`Prometheus`无法直接拉取监控数据，Pushgateway的作用在于提供了中间代理。
    我们可以在应用程序中定时将监控Metrics数据提交到Pushgateway，Prometheus Server定时从Pushgateway的`/metrics`接口采集数据。
5. `Prometheus alerting`: `Prometheus`告警，`Alertmanager`用于管理告警，支持告警通知。
6. `Data visualization`: 数据可视化展示，通过`PromQL`对时间序列数据进行查询、聚合以及逻辑运算。除了Prometheus自带的UI，Grafana对Prometheus也提供了非常好的支持。

Prometheus和其它监控告警方案对比，具有以下几个特点
1. Prometheus属于一站式监控告警平台，依赖少，功能齐全。
2. Prometheus支持对云或容器的监控，其他系统主要对传统物理机监控。
3. Prometheus数据查询语句表现力更强大，内置更强大的统计函数。
4. Prometheus在数据存储扩展性以及持久性上没有InfluxDB，OpenTSDB，Sensu好。

`Prometheus`在设计上放弃了一部分数据准确性（例如2次数据采集间隔数据观察不到），放弃一点准确性得到的是更高可靠性，这里的可靠性提现为架构简单、数据简单、运维简单。

# 三. Prometheus安装
安装Prometheus有三种方法，详见[Prometheus installation](https://prometheus.io/docs/prometheus/latest/installation/)。  
为了方便，我们选择使用Docker安装Prometheus

1. 拉取prom/prometheus镜像  
   $ docker pull prom/prometheus

2. 配置prometheus.yml文件  
   可以配置采集Docker Prometheus容器自身Metrics
   ```
   # 全局设置可以被覆盖
   global:
     # 默认值为15s，用于设置每次数据拉取的间隔
     scrape_interval: 15s
     # 所有时间序列和告警与外部通信时用的外部标签
     external_labels:
       monitor: 'prometheue-monitor'
   
   # scrape配置
   scrape_configs:
   # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
   # jon_name一定要全局唯一
   - job_name: 'prometheus'
     # 覆盖全局的 scrape_interval
     scrape_interval: 5s
     # 静态目标的配置 192.168.0.91为本地物理机IP
     static_configs:
     - targets: ['192.168.0.91:9090']
   ```
  
3. 启动容器  
   1). 使用卷的方式启动  
   $ docker run -p 9090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus  
   2). 也可以基于prom/prometheus这个镜像定制新的镜像  
   ```
   FROM prom/prometheus
   ADD prometheus.yml /etc/prometheus/
   ```
   
   ```
   $ docker build -t my-prometheus .
   $ docker run -p 9090:9090 my-prometheus  
   ```
   
4. 访问Prometheus服务  
   $ http://localhost:9090/
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-localhost.png?raw=true)
   
   同时可以请求`http://localhost:9090/metrics`这个接口获取当前所有的统计指标
   
# 四. Prometheus基础概念
## ① 数据模型
`Prometheus`存储的是`时序数据`，以时间维度存储连续的数据集合。

每个时序数据由三部分组成 `Metric name + Labels + Value`，Metric name和Labels可以唯一标识时序数据。
1. Metric name: 统计指标名称，名称应该具有语义化，例如"fluentd_flowcounter_out_bytes"。Metric name必须满足正则表达式`[a-zA-Z_:][a-zA-Z0-9_:]*`。
2. Labels: 标签是一组Key-Value，labels让Prometheus数据具有多个维度。例如我们可以用{hostname="...", type="..."}这个标签来区分不同的时序数据。标签的命名应该具有语义化，同时必须满足正则表达式`[a-zA-Z_][a-zA-Z0-9_]*`，所有`_`开头的标签名被Prometheus内部保留使用。
3. Value: 实际的时序数据值，包括一个`float64`值和一个毫秒级时间戳。

Prometheus 时序数据可以用以下表达式标识  
`<metric name>{<label name>=<label value>, ...}`

## ② Metrics类型
`Prometheus`有四种Metrics类型，每条时序数据都对应其中一种类型。

1. Counter  
   Counter是一个计数器指标，它是一个只能递增的数值（服务重启的时候会被reset为0）。  
   `Counter主要用于统计服务的请求数、任务完成数、错误出现的次数等`
   
   举例，假设metrics为 http_requests_total
   ```
   1). 总请求: sum(http_requests_total{label1=value1, label2=value2 ...})
   2). 每秒请求量: sum(rate(http_requests_total{label1=value1, label2=value2 ...}[5m]))
   3). 请求量Top 10的URI: topk(10, sum(http_requests_total{label1=value1, label2=value2 ...}) by (path))
   ```

2. Gauge  
   Gauge是一个仪表盘指标，它表示一个既可以递增, 又可以递减的值。  
   `Gauge主要用于统计类似于温度、当前内存使用量等会上下浮动的指标`
   
   举例，假设metrics为 buffer_total_bytes
   ```
   1). 总的buffer: sum(buffer_total_bytes{label1=value1, label2=value2 ...})
   2). buffer按label分类: sum(buffer_total_bytes{label1=value1, label2=value2 ...}) by (label)
   ```

3. Histogram  
   histogram是柱状图，定义一个Histogram类型Metric，则Prometheus系统会自动生成三个对应的指标  
   1). 对每个采样点进行统计，打到各个分类桶中(bucket)  
   2). 对每个采样点值累计和(sum)  
   3). 对采样点的次数累计和(count)  
   
   举例，假设metrics为http_requests_latency_seconds
   ```
   1). 总次数: http_requests_latency_seconds_histogram_count{label1=value1, label2=value2 ...}
   2). 总和: http_requests_latency_seconds_histogram_sum{label1=value1, label2=value2 ...}
   3). 分类桶的数据
       # http请求响应时间 <=0.005 秒的请求次数
       a. http_requests_latency_seconds_histogram_bucket{le="0.005", label1=value1, label2=value2 ...}
       # http请求响应时间 <=0.01 秒的请求次数
       b. http_requests_latency_seconds_histogram_bucket{le="0.01", label1=value1, label2=value2 ...}
   ```
   
4. Summary  
   summary是采样点分位图统计(通常的使用场景：请求持续时间和响应大小)，定义一个Summary类型Metric，则Prometheus系统会自动生成三个对应的指标  
   1). 对于每个采样点进行统计，并形成分位图。    
   2). 统计所有采样点的和  
   3). 统计所有采样点总数  
   
   Summary和Histogram非常类型相似，都可以统计事件发生的次数及其分布情况。Summary和Histogram都提供了对于事件的计数count以及值的汇总sum。 
   因此使用count和sum时间序列可以计算出相同的内容，例如计算 `http每秒的平均响应时间 = rate(basename_sum[5m]) / rate(basename_count[5m])`

   不同在于Histogram可以通过histogram_quantile函数在服务器端计算分位数，而Sumamry的分位数则是直接在客户端进行定义。因此对于分位数的计算，Summary在通过PromQL进行查询时有更好的性能表现，而Histogram则会消耗更多的资源。相对的对于客户端而言Histogram消耗的资源更少。

   举例，例如metrics为http_requests_latency_seconds
   ```
   1). 总次数: http_requests_latency_seconds_summary_count{label1=value1, label2=value2 ...}
   2). 总和: http_requests_latency_seconds_summary_sum{label1=value1, label2=value2 ...}
   3). 分类桶数据
       # http请求响应时间的中位数
       a. http_requests_latency_seconds_summary{label1=value1, label2=value2, quantile="0.5" ...}
       # http请求响应时间的9分位数
       b. http_requests_latency_seconds_summary{label1=value1, label2=value2, quantile="0.9" ...}
   ```

## ③ job和instance
`Prometheus`拉取的每一个目标服务实例称为instance，所有相同服务实例称为`Job`。

例如`api-server`有4个实例
```
job: api-server
  instance 1：1.2.3.4:5670
  instance 2：1.2.3.4:5671
  instance 3：5.6.7.8:5670
  instance 4：5.6.7.8:5671
```

1. `Prometheus`拉取目标服务监控Metric数据的时候，会自动添加`job`和`instance`两个标签。
    * job: 值为prometheus.yml配置的job名称
    * instance: 值为服务实例的`host:port`
2. `Prometheus`会自动生成以下4个Metrics数据
    * `up{job="{job-name}", instance="{host:port}"}`: 这个数据用来反馈目标服务监控状态，值为1表示服务健康，否则表示服务不通。
    * `scrape_duration_seconds{job="{job-name}", instance="{host:port}"}`: 拉取监控Metrics数据时间开销。
    * `scrape_samples_post_metric_relabeling{job="{job-name}", instance="{host:port}"}`: labels变化后，仍然剩余的监控数据条数。
    * `scrape_samples_scraped{job="{job-name}", instance="{host:port}"}`: 拉取监控Metrics数据的条数。

# 五. Prometheus配置
`Prometheus`启动的时候可以加载运行参数`-config.file`指定配置文件，默认为`prometheus.yml`。  
`Prometheus`配置文件是[YAML](http://en.wikipedia.org/wiki/YAML)格式，在配置文件中我们可以指定global, rule_files, scrape_configs, alerting, remote_write, remote_read等属性。

Prometheus.yml配置文件通用格式如下所示
```
global:
   # How frequently to scrape targets by default.
   [ scrape_interval: <duration> | default = 1m ]
  
   # How long until a scrape request times out.
   [ scrape_timeout: <duration> | default = 10s ]
   
   # How frequently to evaluate rules.
   [ evaluation_interval: <duration> | default = 1m ]
   
   # The labels to add to any time series or alerts when communicating with
   # external systems (federation, remote storage, Alertmanager).
   external_labels:
     [ <labelname>: <labelvalue> ... ]
   
# Rule files specifies a list of globs. Rules and alerts are read from
# all matching files.
rule_files:
   [ - <filepath_glob> ... ]

# A list of scrape configurations.
scrape_configs:
   [ - <scrape_config> ... ]
   
# Alerting specifies settings related to the Alertmanager.
alerting:
   alert_relabel_configs:
      [ - <relabel_config> ... ]
   alertmanagers:
      [ - <alertmanager_config> ... ]
   
# Settings related to the experimental remote write feature.
remote_write:
   [ - <remote_write> ... ]
   
# Settings related to the experimental remote read feature.
remote_read:
   [ - <remote_read> ... ]
```

配置文件里面定义了一些占位符，不同占位符含义如下
1. `<boolean>`: 布尔值，值为true或false
2. `<duration>`: 时间字段，必须匹配正则表达式`[0-9]+(ms|[smhdwy])`
3. `<labelname>`: 标签名称，值为字符串必须匹配正则表达式`[a-zA-Z_][a-zA-Z0-9_]*`
4. `<labelvalue>`: 标签值，值为Unicode字符串
5. `<filename>`: 文件路径
6. `<host>`: 主机名
7. `<path>`: URL路径
8. `<scheme>`: HTTP或HTTPS
9. `<string>`: 常规字符串值
10. `<secret>`: 密码字符串
11. `<tmpl_string>`: template扩展的字符串

## ① global
`global`用于`Prometheus`全局默认配置，它主要包含以下4个配置项

1. `scrape_interval`: 拉取目标服务Metrics监控数据时间间隔
2. `scrape_timeout`: 拉取目标服务Metrics监控数据超时时间
3. `evaluation_interval`: 执行rules时间间隔。
4. `external_labels`: 额外的labels，会被添加到每条时序数据上

global配置结构如下
```
global:
   # How frequently to scrape targets by default.
   [ scrape_interval: <duration> | default = 1m ]

   # How long until a scrape request times out.
   [ scrape_timeout: <duration> | default = 10s ]

   # How frequently to evaluate rules.
   [ evaluation_interval: <duration> | default = 1m ]

   # The labels to add to any time series or alerts when communicating with
   # external systems (federation, remote storage, Alertmanager).
   external_labels:
     [ <labelname>: <labelvalue> ... ]
```

配置举例
```
# my global config
global:
   scrape_interval:     15s
   evaluation_interval: 30s
   # scrape_timeout is set to the global default (10s).
   
   external_labels:
     monitor: codelab
     foo:     bar
```

## ② rule_files
`rule_files`用于配置Alert Rules文件列表，支持配置多个文件以及文件目录

rule_files配置结构如下
```
# Rule files specifies a list of globs. Rules and alerts are read from
# all matching files.
rule_files:
   [ - <filepath_glob> ... ]
```

配置举例
```
rule_files:
- "rules/node.rules"
- "rules2/*.rules"
```

## ③ scrape_configs
`scrape_configs`用于配置拉取目标服务Metrics监控数据，每一个拉取配置包含以下配置项

1. `job_name`: 拉取任务名称
2. `scrape_interval`: 拉取时间间隔，没有配置使用global scrape_interval配置
3. `scrape_timeout`: 拉取超时时间，没有配置使用global scrape_timeout配置
4. `metrics_path`: 拉取目标服务Metrics数据的path，默认为/metrics
5. `honor_labels`: 是否要解决labels冲突问题，设置为true以拉取数据为准，否则以配置为准
6. `scheme`: 请求协议，默认为HTTP
7. `params`: 拉取Metrics数据请求参数
8. `basic_auth`: 请求鉴权配置
9. `kubernetes_sd_configs`: K8s服务发现配置
10. `static_configs`: 静态配置目标服务
11. `metric_relabel_configs`: 重置Metrics标签

scrape_configs配置结构如下
```
# A list of scrape configurations.
scrape_configs:
# The job name assigned to scraped metrics by default.
- job_name: <job_name>
  # How frequently to scrape targets from this job.
  [ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

  # Per-scrape timeout when scraping this job.
  [ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

  # The HTTP resource path on which to fetch metrics from targets.
  [ metrics_path: <path> | default = /metrics ]

  # honor_labels controls how Prometheus handles conflicts between labels that are
  # already present in scraped data and labels that Prometheus would attach
  # server-side ("job" and "instance" labels, manually configured target
  # labels, and labels generated by service discovery implementations).
  
  # If honor_labels is set to "true", label conflicts are resolved by keeping label
  # values from the scraped data and ignoring the conflicting server-side labels.
  
  # If honor_labels is set to "false", label conflicts are resolved by renaming
  # conflicting labels in the scraped data to "exported_<original-label>" (for
  # example "exported_instance", "exported_job") and then attaching server-side
  # labels. This is useful for use cases such as federation, where all labels
  # specified in the target should be preserved.
  
  # Note that any globally configured "external_labels" are unaffected by this
  # setting. In communication with external systems, they are always applied only
  # when a time series does not have a given label yet and are ignored otherwise.
    [ honor_labels: <boolean> | default = false ]
    
  # Configures the protocol scheme used for requests.
  [ scheme: <scheme> | default = http ]
    
  # Optional HTTP URL parameters.
  params:
    [ <string>: [<string>, ...] ]
    
  # Sets the `Authorization` header on every scrape request with the
  # configured username and password.
  basic_auth:
    [ username: <string> ]
    [ password: <string> ]
    
  # Sets the `Authorization` header on every scrape request with
  # the configured bearer token. It is mutually exclusive with `bearer_token_file`.
  [ bearer_token: <string> ]
    
  # Sets the `Authorization` header on every scrape request with the bearer token
  # read from the configured file. It is mutually exclusive with `bearer_token`.
  [ bearer_token_file: /path/to/bearer/token/file ]
    
  # Configures the scrape request's TLS settings.
  tls_config:
    [ <tls_config> ]
    
  # Optional proxy URL.
  [ proxy_url: <string> ]

  # List of Kubernetes service discovery configurations.
  # you can set oterh sd configs
  kubernetes_sd_configs:
    [ - <kubernetes_sd_config> ... ]

  # List of labeled statically configured targets for this job.
  static_configs:
    [ - <static_config> ... ]
    
  # List of target relabel configurations.
  relabel_configs:
    [ - <relabel_config> ... ]
    
  # List of metric relabel configurations.
  metric_relabel_configs:
    [ - <relabel_config> ... ]
    
  # Per-scrape limit on number of scraped samples that will be accepted.
  # If more than this number of samples are present after metric relabelling
  # the entire scrape will be treated as failed. 0 means no limit.
  [ sample_limit: <int> | default = 0 ]
```

配置举例
```
scrape_configs:
- job_name: 'prometheus'
  scrape_interval: 5s
  static_configs:
  - targets: ['127.0.0.1:9090']

- job_name: 'node'
  scrape_interval: 8s
  static_configs:
  - targets: ['127.0.0.1:9100', '127.0.0.12:9100']

- job_name: 'mysqld'
  static_configs:
  - targets: ['127.0.0.1:9104']
      
- job_name: 'memcached'
  static_configs:
  - targets: ['127.0.0.1:9150']
```

## ④ alerting
`alerting`用于告警配置，主要有以下2个配置项

1. `alert_relabel_configs`: 动态修改alert的配置规则
2. `alertmanagers`: 动态发现Alertmanager的配置

alerting配置结构如下
```
# Alerting specifies settings related to the Alertmanager.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]
```

配置举例
```
alerting:
  alertmanagers:
  - scheme: https
    static_configs:
    - targets:
      - "1.2.3.4:9093"
      - "1.2.3.5:9093"
      - "1.2.3.6:9093"
```

## ⑤ 配置举例
`Prometheus`配置参数比较多，使用较多的是 global, rule_files 和scrape_configs等。

```
# my global config
global:
  scrape_interval:     15s
  evaluation_interval: 30s
  # scrape_timeout is set to the global default (10s).

  external_labels:
    key1: value1
    key2: value2

rule_files:
- "first.rules"
- "my/*.rules"

scrape_configs:
# job prometheus 静态配置
- job_name: 'prometheus'
  scrape_interval: 5s
  static_configs:
  - targets: ['127.0.0.1:9090']
      
# job node 静态配置
- job_name: 'node'
  scrape_interval: 8s
  static_configs:
  - targets: ['127.0.0.1:9100', '127.0.0.12:9100']
  
# job mysqld 静态配置
- job_name: 'mysqld'
  static_configs:
  - targets: ['127.0.0.1:9104']

# job memcached 静态配置
- job_name: 'memcached'
  static_configs:
  - targets: ['127.0.0.1:9150']
  
# job service-kubernetes 服务发现
- job_name: service-kubernetes
  kubernetes_sd_configs:
  - role: endpoints
    api_server: 'https://localhost:1234'
    basic_auth:
       username: 'myusername'
       password: 'mysecret'

# job service-kubernetes-namespaces 服务发现
- job_name: service-kubernetes-namespaces
  kubernetes_sd_configs:
  - role: endpoints
    api_server: 'https://localhost:1234'
    namespaces:
       names:
       - default

alerting:
  alertmanagers:
  - scheme: https
    static_configs:
    - targets:
      - "1.2.3.4:9093"
      - "1.2.3.5:9093"
      - "1.2.3.6:9093"
```

# 六. Prometheus查询
`Prometheus`提供了内置的`PromQL`，用户可以实时的查询和聚合时序数据，计算结果可以通过Expression Browser、Grafana展示，也可以通过HTTP API请求获取。

`PromQL`查询结果有4种类型
1. `Instant vector`: 即时向量，同一时间点的一组时序数据
2. `Range vector`: 范围向量，一个时间段内的一组时序数据
3. `Scalar`: 标量，一个浮点数值
4. `String`: 一个字符串，目前未使用

为了方便演示Prometheus查询，我们假设Prometheus每5s拉取一次数据，所有Metrics数据均为counter类型，当前时间戳为`1551769823`，当前有以下Metrics数据。`不同的Metrics类型 统计语法不同，以下查询示例都是基于counter来演示`

| Type | Metric | Labels | Value | Timestamp |
| ------ |------ | ------ | ------ | ------ |
| counter | http_requests_total | {job="prometheus",group="canary",environment="test",method="Get"} | 5.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="test",method="Post"} | 4.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="test",method="Put"} | 3.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Get"} | 10.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Post"} | 8.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Put"} | 6.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="release",method="Get"} | 20.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="release",method="Post"} | 16.0 | @1551769822.89 |
| | | {job="prometheus",group="canary",environment="release",method="Put"} | 12.0 | @1551769822.89 |
| counter |http_requests_total | {job="prometheus",group="canary",environment="test",method="Get"} | 4.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="test",method="Post"} | 3.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="test",method="Put"} | 2.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Get"} | 9.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Post"} | 7.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Put"} | 5.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="release",method="Get"} | 19.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="release",method="Post"} | 15.0 | @1551769817.89 |
| | | {job="prometheus",group="canary",environment="release",method="Put"} | 11.0 | @1551769817.89 |
| counter |http_requests_total | {job="prometheus",group="canary",environment="test",method="Get"} | 3.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="test",method="Post"} | 3.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="test",method="Put"} | 2.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Get"} | 8.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Post"} | 6.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="beta",method="Put"} | 4.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="release",method="Get"} | 18.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="release",method="Post"} | 14.0 | @1551769812.89 |
| | | {job="prometheus",group="canary",environment="release",method="Put"} | 10.0 | @1551769812.89 |


## ① 默认查询  
   查询http_requests_total所有数据: `http_requests_total`
   ```
   {job="prometheus",group="canary",environment="test",method="Get"}       5.0
   {job="prometheus",group="canary",environment="test",method="Post"}      4.0
   {job="prometheus",group="canary",environment="test",method="Put"}       3.0
   {job="prometheus",group="canary",environment="beta",method="Get"}       10.0
   {job="prometheus",group="canary",environment="beta",method="Post"}      8.0
   {job="prometheus",group="canary",environment="beta",method="Put"}       6.0
   {job="prometheus",group="canary",environment="release",method="Get"}    20.0
   {job="prometheus",group="canary",environment="release",method="Post"}   16.0
   {job="prometheus",group="canary",environment="release",method="Put"}    12.0
   ```

## ② 条件查询  
   PromQL支持4种label条件表达式
   > `=`: 等于条件，例如 http_requests_total{environment="test"}  
   > `!=`: 不等于条件，例如 http_requests_total{environment!="test"}  
   > `=~`: 匹配正则表达式条件，例如 http_requests_total{environment=~"t.*"}  
   > `!~`: 不匹配正则表达式条件，例如 http_requests_total{environment!=~"t.*"}  
    
   1. 查询http_requests_total environment label值为test的时序数据  
   `http_requests_total{environment="test"}`
   ```
   {job="prometheus",group="canary",environment="test",method="Get"}       5.0
   {job="prometheus",group="canary",environment="test",method="Post"}      4.0
   {job="prometheus",group="canary",environment="test",method="Put"}       3.0
   ```
   2. 查询http_requests_total environment label值为test, method label值为Get的时序数据  
   `http_requests_total{environment="test",method="Get"}`
   ```
   {job="prometheus",group="canary",environment="test",method="Get"}       5.0
   ```
   
## ③ 模糊查询  
   1. 查询http_requests_total environment label值匹配`t.*`正则的时序数据  
   `http_requests_total{environment=~"t.*"}`
   ```
   {job="prometheus",group="canary",environment="test",method="Get"}       5.0
   {job="prometheus",group="canary",environment="test",method="Post"}      4.0
   {job="prometheus",group="canary",environment="test",method="Put"}       3.0
   ```
   
   2. 查询http_requests_total environment label值不匹配`t.*`正则的时序数据  
   `http_requests_total{environment!~"t.*"}`
   ```
   {job="prometheus",group="canary",environment="beta",method="Get"}       10.0
   {job="prometheus",group="canary",environment="beta",method="Post"}      8.0
   {job="prometheus",group="canary",environment="beta",method="Put"}       6.0
   {job="prometheus",group="canary",environment="release",method="Get"}    20.0
   {job="prometheus",group="canary",environment="release",method="Post"}   16.0
   {job="prometheus",group="canary",environment="release",method="Put"}    12.0
   ```
  
## ④ 区间查询
   Prometheus支持在查询条件后通过`[]`指定，由数字和单位组成，支持以下几种类型单位
   > `s`: 秒  
   > `m`: 分  
   > `h`: 时  
   > `d`: 天  
   > `w`: 周  
   > `y`: 年  

   1. 查询http_requests_total 最近10s environment label值为test的数据  
   `http_requests_total{environment="test"}[10s]`
   ```
   {job="prometheus",group="canary",environment="test",method="Get"}   5.0  @1551769822.89
                                                                       4.0  @1551769817.89 
   {job="prometheus",group="canary",environment="test",method="Post"}  4.0  @1551769822.89
                                                                       3.0  @1551769817.89
   {job="prometheus",group="canary",environment="test",method="Put"}   3.0  @1551769822.89
                                                                       2.0  @1551769817.89
   ```
   
   2. 查询http_requests_total 最近10s environment label值为test，method label值为Get的数据  
   `http_requests_total{environment="test", method="Get"}[10s]`
   ```
   {job="prometheus",group="canary",environment="test",method="Get"}   5.0  @1551769822.89
                                                                       4.0  @1551769817.89
   ```
   
## ⑤. 聚合查询
   Prometheus内置支持以下几种类型的聚合函数，用于即时向量聚合操作
   > `sum`: 求和  
   > `min`: 求最小值  
   > `max`: 求最大值  
   > `avg`: 求平均值  
   > `stddev`: 求标准差  
   > `stdvar`: 求标准差异  
   > `count`: 计数  
   > `count_values`: 相同数据值计数  
   > `bottomk`: 最小k个数  
   > `topk`: 最大k个数  
   > `quantile`: 分布统计  
    
   1. 求http_requests_total 所有指标和  
   `sum(http_requests_total){environment="test"}`
   ```
   29
   ```

   2. 统计http_requests_total 所有记录数   
   `count(http_requests_total{environment="test"})`
   ```
   9
   ```
   
   3. 计算http_requests_total 具有相同Value的记录  
   `count_values("total_request",http_requests_total{environment="test"})`
   ```
   {total_request="5.0"}   1
   {total_request="4.0"}   1
   {total_request="3.0"}   1
   ```
   
   4. 计算http_requests_total top3指标  
   `topk(3, http_requests_total)`
   ```
   {job="prometheus",group="canary",environment="release",method="Get"}    20.0
   {job="prometheus",group="canary",environment="release",method="Post"}   16.0
   {job="prometheus",group="canary",environment="release",method="Put"}    12.0
   ```
   
## ⑥ 函数计算
   除了上面提到的聚合函数外，Prometheus有很多内置函数[Prometheus functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)，常用的函数有以下几个  
   > `rate(range vector)`: 用于计算过去一段时间每秒平均值，取指定时间范围内所有数据点，算出一组速率，然后取平均值作为结果。仅适用于counter类型数据，适合缓慢变化的数据分析。  
   > `irate(range vector)`: 用于计算过去一段时间每秒平均值，取指定时间范围内最近两个数据点来算速率，然后作为结果。仅适用于counter类型数据，适合快速变化的数据分析。 
    
   1. 分析http_requests_total指标变化，常用在Grafana上统计某一个指标的趋势分析  
   `sum(rate(http_requests_total{environment="release"}[2m]))`  
   `sum(rate(http_requests_total{environment="release"}[2m])) by (method)`
    
# 七. Prometheus可视化
  Prometheus默认有Expression Browser进行数据展示，当Prometheus启动后访问`http://host:9090/graph`即可进行数据查询
  
  ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-localhost.png?raw=true)
  
  1. `Alerts`: 告警相关的配置
  2. `Graph`: 数据查询、展示
  3. `Status`: Prometheus服务运行状态、配置以及服务发现等
  

