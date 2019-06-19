---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Fluentd
---

# 一. 介绍
Fluentd做为一个开源的日志采集器，专为处理日志流而设计。  
目前，应用程序部署环境基本可以分为3类，针对这3类Fluentd采集日志的方案是不一样的。

1. 物理机环境: 最原始的部署方式，每个应用负责把日志写到指定目录文件。每台物理机部署一个Fluentd应用，Fluentd配置从指定目录进行采集。
2. Docker: Docker run方式启动容器，容器内部署应用，应用日志可以写标准输出也可以写文件。可以通过卷挂载的方式，让容器日志输出到宿主机上，Fluentd配置从指定目录进行采集。
3. K8s: 应用部署在K8s Pod上，应用日志可以标准输出也可以写文件。推荐做法是Pod内容器通过stdout和stderr流输出日志，docker log-driver会把日志写到Node的/var/log/pods目录下，K8s默认会在每个Node上创建/var/log/containers/xxx.log软链接到/var/log/pods目录下log文件。只需要在K8s每个Node部署一个Fluentd ds，Fluentd配置采集Node的/var/log/containers即可。

本文会简单介绍，Fluentd日志采集全链路方案，实际在生产过程中会有很多的细节不一样，但是大致的流程是一致的。

# 二. 方案
日志处理全链路包括以下几个部分: 日志生产、日志采集、日志处理、日志展示，日志监控，整体的方案如下

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd_host_pipeline.png?raw=true)

1. 日志生产: 应用程序部署在物理机、docker容器或者K8s，每个应用都会生产日志
2. 日志采集: 采用Fluentd来采集日志，采集完成后写入Kafka
3. 日志处理: 采用Logstash消费Kafka日志数据，再写入Elasticsearch
4. 日志展示: 数据存储在Elasticsearch，采用Kibana做为前端查询工具
5. 日志监控: 使用Prometheus监控Fluentd，并在Grafana进行报表展示

# 三. 部署
为了方便，所有部署我们采用Docker的方式。

## ① Kafka
由于Fluentd采集完日志后需要先写入Kafka，我们需要先部署Kafka环境。

1. 镜像下载  
   `$ docker pull wurstmeister/kafka`  
   `$ docker pull zookeeper`
2. 创建一个新的bridge网络  (后续所有的容器都使用这个网络进行连接)
   `$ docker network create docker-net`
3. 启动Zookeeper  
   `$ docker run -d --name zookeeper --restart always --network docker-net zookeeper`
4. 启动Kafka  
   `$ docker run -d --name kafka --network docker-net -e KAFKA_ADVERTISED_HOST_NAME=kafka -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 wurstmeister/kafka`
5. 确认Zookeeper和Kafka是否启动成功  
   `$ docker ps -a | grep zoo`  
   `$ docker ps -a | grep kafka`
6. 创建topic  
   `$ docker run --rm --network docker-net wurstmeister/kafka \kafka-topics.sh --create --topic host_http_api_biz --replication-factor 1 --partitions 1 --zookeeper zookeeper:2181`  
   `$ docker run --rm --network kafka-net wurstmeister/kafka \kafka-topics.sh --create --topic host_http_api_access --replication-factor 1 --partitions 1 --zookeeper zookeeper:2181`  
   `$ docker run --rm --network kafka-net wurstmeister/kafka \kafka-topics.sh --create --topic host_var_log  --replication-factor 1 --partitions 1 --zookeeper zookeeper:2181`

## ② Fluentd
Fluentd负责采集原始业务日志，并把数据写入Kafka

1. 下载镜像  
   `$ docker pull fluent/fluentd:v0.12.30`  (指定版本为v0.12.30)
2. 自定义镜像   
   `$ docker build -t fluent/fluentd-cgl:v1.0.0-1`
   ```
   # base image
   FROM fluent/fluentd:v0.12.30

   # MAINTAINER 维护者信息
   MAINTAINER chenguolin

   # install fluent-plugin-elasticsearch
   RUN gem install 'fluent-plugin-copy_ex' -v '~>0.1.0' \
      && gem install 'fluent-plugin-kafka' -v '~>0.6.0' \
      && gem install 'ruby-kafka' -v '~>0.6.8' \
      && gem install 'fluent-plugin-prometheus' -v '~>0.3.0' \
      && gem uninstall 'ruby-kafka' -v '~>0.7.6'

   # 拷贝fluent.conf
   COPY fluent.conf /fluentd/etc
   ```
3. 运行Fluentd  
   `$ docker run -d --name fluentd --network docker-net --privileged=true -v /var/log:/var/log -v /www:/www fluent/fluentd-cgl:v1.0.0-6`  
   1). --network docker-net: 网络指的为docker-net保证能够连接到kafka brokers  
   2). -v 参数: 物理机目录挂载，保证容器内能够访问到
 
   fluent.conf配置文件如下
   ```
   # 采集/var/log目录下日志
   <source>
     @type tail
     tag var_log
     path /var/log/*.log
     pos_file /www/fluentd/var.log.pos
     read_from_head true
     format none
   </source>

   # 采集/www/http-api/logs/go-http-api.log
   <source>
     @type tail
     tag http_api_biz
     path /www/http-api/logs/go-http-api.log
     pos_file /www/fluentd/http-api-biz.log.pos
     read_from_head true
     format none
   </source>

   # 采集/www/http-api/logs/access.log
   <source>
     @type tail
     tag http_api_access
     path /www/http-api/logs/access.log
     pos_file /www/fluentd/http-api-access.log.pos
     read_from_head true
     format none
   </source>

   # 添加var_log数据kafka topic
   <filter var_log.**>
     @type record_transformer
     <record>
        topic host_var_log
     </record>
   </filter>

   # 添加http_api_biz数据kafka topic
   <filter http_api_biz.**>
     @type record_transformer
     <record>
       topic host_http_api_biz
     </record>
   </filter>

   # 添加http_api_access数据kafka topic
   <filter http_api_access.**>
     @type record_transformer
     <record>
       topic host_http_api_access
     </record>
   </filter>

   # 输出
   <match **>
      @type copy_ex
      <store ignore_error>
         @type kafka_buffered
         # broker地址 如果有多个按逗号分隔 
         brokers 192.168.0.91:9092 
         default_topic fluentd_unexpected-logs
         get_kafka_client_log true

         output_include_tag false
         output_include_time false
         exclude_topic_key false
         exclude_partition_key true

         output_data_type json

         buffer_type file
         buffer_path /www/fluentd/buffer/host
         buffer_chunk_limit 64M
         buffer_queue_limit 512
         flush_interval 3s

         kafka_agg_max_bytes 1M
         max_send_limit_bytes 1M
         compression_codec gzip
         required_acks 1
         num_threads 4
         max_send_retries 3
         discard_kafka_delivery_failed false
         disable_retry_limit true
         max_retry_wait 10
     </store>
   </match>

   # prometheus监控配置
   # input_prometheus 插件提供HTTP接口供Prometheus拉取监控数据
   <source>
     @type prometheus
     bind 0.0.0.0
     port 24231
     metrics_path /metrics
   </source>

   # input_prometheus_output_monitor 插件监控output插件
   <source>
     @type prometheus_output_monitor
     interval 5
     <labels>
       name fluentd_host_log
     </labels>
   </source>

   # filter_prometheus 统计总的input records
   <filter **>
     @type prometheus
     <metric>
       name fluentd_input_records
       type counter
       desc The total number of incoming records
     </metric>
   </filter>
   ```
   
3. 查看Fluentd日志是否正常
   `$ docker logs -f fluentd`

4. 查看Kafka数据  
   `$ docker run --rm --network docker-net wurstmeister/kafka \kafka-console-consumer.sh --topic host_var_log --from-beginning --bootstrap-server kafka:9092 --group console-group`  
   `$ docker run --rm --network docker-net wurstmeister/kafka \kafka-console-consumer.sh --topic host_http_api_biz --from-beginning --bootstrap-server kafka:9092 --group console-group`  
   `$ docker run --rm --network docker-net wurstmeister/kafka \kafka-console-consumer.sh --topic host_http_api_access --from-beginning --bootstrap-server kafka:9092 --group console-group` 

## ③ Elasticsearch
Elasticsearch是一个开源的搜索引擎。

1. 下载镜像  
   `$ docker pull docker.elastic.co/elasticsearch/elasticsearch:6.6.2`
2. 运行Elasticsearch  
   `$ docker run -d --name elasticsearch --net docker-net -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.6.2`
3. 查看elasticsearch日志  
   `$ docker logs -f elasticsearch`

## ④ Logstash
Logstash负责从Kafka消费数据，并把数据写入Elasticsearch

1. 下载镜像  
   `$ docker pull docker.elastic.co/logstash/logstash:6.6.2`
2. 启动Logstash  
   `docker run -d --name logstash --net docker-net  -v ~/k8s/tmp/logstash:/config-dir docker.elastic.co/logstash/logstash:6.6.2 -f /config-dir/logstash.conf`
   
   logstash.conf配置文件如下，支持不同topic写到不同的es index，并按天区分 [logstash读取kafka输出到es里怎么根据不同的topic打到不同的索引](https://www.zhihu.com/question/276783606)
   ```
   input{
      kafka{
         bootstrap_servers => ["172.19.0.3:9092"]
         topics_pattern => ".*"
         decorate_events => true
         codec => json {
            charset => "UTF-8"
        }
      }
   }
   output{
      elasticsearch{
        hosts => "172.19.0.5:9200"
        index => "%{[@metadata][kafka][topic]}-%{+YYYY.MM.dd}"
      }
   }
   ```
3. 查看logstash日志  
   `$ docker logs -f logstash`

## ⑤ Kibana
Kibana是一个针对Elasticsearch的开源分析及可视化平台，用来搜索、查看交互存储在Elasticsearch索引中的数据。 使用Kibana，可以通过各种图表进行高级数据分析及展示。

1. 下载镜像  
   `$ docker pull docker.elastic.co/kibana/kibana:6.6.2`
2. 启动Kibana  
   `$ docker run -d --name kibana --net docker-net -p 5601:5601 -e "ELASTICSEARCH_URL=http://elasticsearch:9200" docker.elastic.co/kibana/kibana:6.6.2`
3. 查看日志  
   `$ docker logs -f kibana`
4. 数据查询`http://localhost:5601`
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kibana_create_index_pattern.png?raw=true)
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kibana_discover_data.png?raw=true)

## ⑥ Prometheus
Prometheus是SoundCloud开源的监控告警解决方案

1. 下载镜像  
   `$ docker pull prom/prometheus:v2.8.0`
2. 启动prometheus    
   `$ docker run -d --name prometheus --net docker-net -p 9090:9090 -v ~/k8s/tmp/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus:v2.8.0`
   
   prometheus.yml配置文件
   ```
   global:
     scrape_interval: 15s
     external_labels:
       monitor: 'prometheue-monitor'

   scrape_configs:
   - job_name: 'fluentd'
     scrape_interval: 5s
     static_configs:
     - targets: ['172.19.0.4:24231']
   ```
3. 查看prometheus日志  
   `$ docker logs -f prometheus`
4. 查看prometheus监控数据`http://localhost:9090`
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-localhost.png?raw=true)

## ⑦ Grafana
Grafana是一个开源仪表盘工具，它可用于Graphite、InfluxDB与 OpenTSDB 一起使用。最新的版本还可以用于其他的数据源，比如 Elasticsearch。 从本质上说，它是一个功能丰富的Graphite-web 替代品，能帮助用户更简单地创建和编辑仪表盘。它包含一个独一无二的 Graphite 目标解析器，从而可以简化度量和函数的编辑。Grafana 快速的客户端渲染默认使用的是 Flot ，即使很长的时间范围也可应对，这样用户就可以创建具有智能轴格式（比如线和点）的复杂图表了。

1. 下载镜像  
   `$ docker pull grafana/grafana:5.0.0`
2. 启动Grafana  
   `$ docker run -d --name grafana --net docker-net -p 3000:3000 grafana/grafana:5.0.0`
3. 查看grafana日志  
   `$ docker logs -f grafana`
4. 使用grafana展示监控数据`http://localhost:3000`
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-prometheus.png?raw=true)
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_buffer_queue_length.png?raw=true)
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana_fluentd_output_status_buffer_total_bytes.png?raw=true)

