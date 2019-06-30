---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
---

# 一. Install
启动运行Prometheus主要有两种方式，传统的物理机方式启动和Docker方式启动

1. 物理机方式启动Prometheus  
```
1). 从https://prometheus.io/download/ 下载Prometheus对应的tar.gz
2). 解压: tar -xvfz prometheus-*.tar.gz
3). 配置Prometheus
    a. cd prometheus-*
    b. 编辑 prometheus.yml
    c. 校验 prometheus.yml: ./promtool check config prometheus.yml
4). 启动Prometheus: ./prometheus --config.file=prometheus.yml
5). 查看Metrics: http://localhost:9090/graph
```
   
2. Docker方式启动Prometheus  
```
1). 拉取prom/prometheus镜像: docker pull prom/prometheus
2). 配置prometheus.yml
3). 校验 prometheus.yml: ./promtool check config prometheus.yml
4). 启动Prometheus容器: docker run -p 9090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
5). 查看Metrics: http://localhost:9090/graph
```
   
3. Prometheus支持Reload配置，按照以下方式即可  
```
1). kill -HUP {pid}
2). curl -X POST http://localhost:9090/-/reload
```

# 二. Metrics
Prometheus根本上把所有的监控数据存储为`时序数据`，每个时序数据由`metric`名称和一序列`labels`组成，以及一个float64类型数值

## ① Metrics
1. metrics名称必须满足 `[a-zA-Z_:][a-zA-Z0-9_:]*` 这个正则表达式，label名称必须满足 `[a-zA-Z_][a-zA-Z0-9_]*` 这个正则表达式
2. metrics名称需要使用前缀来表示特定的服务，例如`prometheus`_notifications_total表示Prometheus服务，`http`_request_duration_seconds表示HTTP服务
3. metrics名称需要使用后缀来表示监控数值的单位，例如http_request_duration_`seconds`和node_memory_usage_`bytes`，http_requests_`total` total则是用来表示次数
4. metrics名称应该使用基本单位，例如seconds，bytes，meters
5. metrics名称最好使用单个单位，不要使用多个单位，例如不要混淆seconds 和 bytes
6. 不应该把label名称放在metrics名称上   
    
## ② Type
1. Counter: Counter类型metrics用于统计数值只可以增加的指标，一般使用Counter类型metrics统计例如`请求数`、`任务完成数`、`错误数`等，当服务重启的时候数值会被重置为0
2. Gauge: Gauge类型metrics用于统计数值可以上下浮动的指标，一般使用Gauge类型metrics统计例如`温度`、`内存使用`等，当服务重启的时候数值会被重置为0
3. Histogram: Histogram类型metrics用于对监控结果进行采样，并支持在可配置的桶上计数，同时提供了所有监控数值的和，例如用于统计`请求时间`、`响应大小`等
4. Summary: Summary和Histogram类似，它还提供在滑动窗口上配置分位数   
    
## ③ Automatically generated
1. Prometheus拉取目标服务metrics的时候，会在每个metrics上自动添加2个label `job`和`instance`
    * `job`: prometheus.yml配置的拉取目标服务的名称
    * `instance`: 具体拉取的目标服务地址，数值格式为<host>:<port>
2. Prometheus拉取目标服务metrics的时候，会自动带上以下4个metrics
    * `up{job="<job-name>", instance="<instance-id>"}`: 表示instance健康状态，1表示健康，0表示拉取失败
    * `scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}`: 表示拉取metrics时间周期
    * `scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}`: 表示relabel后仍然剩余的监控数据条数
    * `scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}`: 表示拉取metrics数据的条数

# 三. Usage
## ① Configuration  
Prometheus在启动的时候可以通过`--config.file`设置prometheus.yml配置文件  
一个有效的prometheus.yml例如可以参考 https://github.com/prometheus/prometheus/blob/release-2.9/config/testdata/conf.good.yml  
   
prometheus.yml支持监控Prometheus自身服务，具体可以参考以下配置
```
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

  # Override the global default and scrape targets from this job every 5 seconds.
  scrape_interval: 5s

  static_configs:
    - targets: ['localhost:9090']
```
   
## ② Querying  
1. Prometheus提供了丰富的操作运算符，具体可以参考 https://prometheus.io/docs/prometheus/latest/querying/operators/  
2. Prometheus提供了丰富的运算函数，具体可以参考 https://prometheus.io/docs/prometheus/latest/querying/functions/  
3. Prometheus支持使用HTTP API查询方式查询数据，具体可以参考 https://prometheus.io/docs/prometheus/latest/querying/api/  
4. 查询举例  
    * 查询当前http_requests_total时序数据: `http_requests_total{job=".*server", handler="/api/comments"}`
    * 查询最近5分钟http_requests_total时序数据: `http_requests_total{job=".*server", handler="/api/comments"}[5m]`
    * 查询最近5分钟http_requests_total每秒变化率: `rate(http_requests_total[5m])`
    * 查询最近5分钟http_requests_total每秒变化率和并按job分组: `sum(rate(http_requests_total[5m])) by (job)`
    * 查询最近5分钟instance_cpu_time_ns每秒变化率和并按app和proc分组后的top 3: `topk(3, sum(rate(instance_cpu_time_ns[5m])) by (app, proc))`
    * 查询instance_cpu_time_ns时序数据条数并按app分组: `count(instance_cpu_time_ns) by (app)`

## ③ Storage  
1. Prometheus支持把时序数据存储在本地存储和远程存储介质，我们重点介绍一下本地存储的实现  
2. 每2个小时数据放在同一个block，一个block包含一个或多个chunk文件，当前block数据是在内存里面 `这里存在一个问题: 如果当前2个小时数据很多，会导致Prometheus server服务性能变差，在Grafana上查询的时候会很慢很卡`  
3. Prometheus存储数据目录结构如下所示
```
./data/01BKGV7JBM69T2G1BGBGM6KB12
./data/01BKGV7JBM69T2G1BGBGM6KB12/meta.json
./data/01BKGTZQ1SYQJTR4PB43C8PD98
./data/01BKGTZQ1SYQJTR4PB43C8PD98/meta.json
./data/01BKGTZQ1SYQJTR4PB43C8PD98/index
./data/01BKGTZQ1SYQJTR4PB43C8PD98/chunks
./data/01BKGTZQ1SYQJTR4PB43C8PD98/chunks/000001
./data/01BKGTZQ1SYQJTR4PB43C8PD98/tombstones
./data/01BKGTZQ1HHWHV8FBJXW1Y3W0K
./data/01BKGTZQ1HHWHV8FBJXW1Y3W0K/meta.json
./data/01BKGV7JC0RY8A6MACW02A2PJD
./data/01BKGV7JC0RY8A6MACW02A2PJD/meta.json
./data/01BKGV7JC0RY8A6MACW02A2PJD/index
./data/01BKGV7JC0RY8A6MACW02A2PJD/chunks
./data/01BKGV7JC0RY8A6MACW02A2PJD/chunks/000001
./data/01BKGV7JC0RY8A6MACW02A2PJD/tombstones
./data/wal/00000000
./data/wal/00000001
./data/wal/00000002
```
      
## ④ Alert  
Prometheus根据Alerting rules发送告警到`Alertmanager`，Alertmanager管理这些报警，最终通过Email、PagerDuty、HipChat等方式发送通知  
Prometheus设置监控和告警需要以下3步
   
1. Alertmanager需要配置一个配置文件，具体可以参考 https://prometheus.io/docs/alerting/configuration/  
   一个有效的配置文件例如 https://github.com/prometheus/alertmanager/blob/master/doc/examples/simple.yml  
   
2. prometheus.yml配置Prometheus和Alertmanager通信  
   ```
   alerting:
      alert_relabel_configs:
        [ - <relabel_config> ... ]
      alertmanagers:
        - scheme: https
          static_configs:
          - targets:
            - "1.2.3.4:9093"
            - "1.2.3.5:9093"
            - "1.2.3.6:9093"
   ```
   
3. 定义告警规则  
   参考 https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/  
   
# 三. Visualization
Prometheus有2种常见的可视化选择方案

1. Expression browser
   Prometheus服务默认的可视化，Prometheus服务启动之后用户只需要访问 http://host:9090/graph 即可进行数据查询
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-localhost.png?raw=true)
   
2. Grafana  
   Grafana支持Prometheus查询，推荐使用Grafana，具体可以参考 https://prometheus.io/docs/visualization/grafana/
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-k8s-monitor.png?raw=true)

# 四. Best Practices
1. Metric and label naming: https://prometheus.io/docs/practices/naming/
2. Instrument: https://prometheus.io/docs/practices/instrumentation/
3. Histograms and summaries: https://prometheus.io/docs/practices/histograms/
4. Alerting: https://prometheus.io/docs/practices/alerting/
5. Recording Rules: https://prometheus.io/docs/practices/rules/
6. using the Pushgateway: https://prometheus.io/docs/practices/pushing/
7. Monitoring Linux host metrics: https://prometheus.io/docs/guides/node-exporter/
8. Instrument Go applications: https://prometheus.io/docs/guides/go-application/
9. Monitoring Docker container metrics: https://prometheus.io/docs/guides/cadvisor/
10. file-based service discovery: https://prometheus.io/docs/guides/file-sd/


