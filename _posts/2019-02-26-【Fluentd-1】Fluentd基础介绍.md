---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Fluentd
---

# 一. Fluentd介绍
`Fluentd`是一个开源的数据收集器，专为处理数据流设计，使用`JSON`作为数据格式。它采用了插件式的架构，具有高可扩展性、高可用性，同时还实现了高可靠的消息转发。`Fluentd`是由Fluent+d得来，d形象地标明了它是以一个守护进程的方式运行。  
我们可以把各种不同来源的数据，首先发送给Fluentd，接着Fluentd根据配置通过不同的插件把数据转发到不同的地方，比如文件、数据库甚至可以转发到另一个Fluentd。

`Fluentd目前主要用作 统一日志收集器，支持各种不同系统的日志收集处理。`

Fluentd由Ruby和C语言编写的，Fluentd现在有两个版本`v0.12`和`v1.0`，`0.12`版本是目前使用最多的稳定版本，`v1.0`版本是目前新的稳定版本拥有新的plugin API。


## ① 传统日志系统
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-old-log-architecture.jpg?raw=true)

如上图所示，传统的日志系统非常复杂，不同业务之间耦合度非常高，这就带来了几个问题。  
1. 系统复杂，开发运维成本高
2. 业务耦合度高，业务需要做各种定制比较耗时

## ② 使用Fluentd日志系统
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-new-architecture.png?raw=true)

`Fluentd`提出统一日志层的概念，基本思想是通过Fluentd统一处理日志数据的各个方面，收集、过滤、缓冲和输出。  
使用Fluentd后解决了传统日志系统的2个问题，生产日志的业务只需要把日志流给Fluentd，通过Fluentd处理最终输出到不同地方进行存储消费。

Fluentd有以下几个特点
1. 使用JSON做为数据记录: Fluentd尽可能地将数据结构转化为JSON格式方便统一处理。
   ![](https://www.fluentd.org/assets/img/architecture/log-as-json.png)
2. 可插拔架构: Fluentd有一个灵活的插件系统，目前社区提供了500多个的插件，利用这些插件我们可以更好的处理日志消息流。
   ![](https://www.fluentd.org/assets/img/architecture/pluggable.png)
3. 最小的资源需求: Fluentd是用C语言和Ruby语言编写的，只需要很少的系统资源。
   ![](https://www.fluentd.org/assets/img/architecture/c-and-ruby.png)
4. 内置可靠性: Fluentd支持基于内存和文件的缓冲，以防止数据丢失。同时Fluentd还支持多点备份，保证高可用。
   ![](https://www.fluentd.org/assets/img/architecture/reliable.png)

# 二. Fluentd安装
我们采用Docker方式来安装Fluentd，也可以采用其它方式安装例如`Ruby Gem`，具体可以参考文档[Fluentd installation](https://docs.fluentd.org/v0.12/categories/installation)

下面介绍如何使用Docker安装Fluentd，并通过Fluentd收集日志存储到Elasticsearch。

1. 下载fluentd镜像  
   $ docker pull fluent/fluentd:latest

2. 编写fluentd.conf配置文件，可以使用`fluentd --dry-run -c fluentd.conf`校验配置文件是否正确
   ```
   # get logs from docker-ce log
   <source>
     @type tail
     <parse>
       @type syslog
       # parse section parameters
       time_format %Y-%m-%dT%H:%M:%SZ
     </parse>
     port 42185
     path /var/log/docker-ce.log
     tag docker_ce_logs
   </source>
   
   # get logs from fluent-logger, fluent-cat or other fluentd instances
   <source>
     @type forward
   </source>
   
   <match **>
     @type elasticsearch
     host "elasticsearch"
     logstash_format true
     <buffer>
       flush_interval 10s # for testing
     </buffer>
   </match>
   ```

3. 基于fluent/fluentd:latest基础镜像构建安装fluent-plugin-elasticsearch插件新镜像
   ```
   # base image
   FROM fluent/fluentd:latest
   
   # MAINTAINER 维护者信息
   MAINTAINER chenguolin
   
   # install fluent-plugin-elasticsearch
   RUN gem install fluent-plugin-elasticsearch --no-document
   ```  
   $ docker build --tag=fluent/fluentd/cgl-test:v0.0.1-1

4. 运行fluentd  
   $ docker run -d -p 42185:42185 -v /var/log:/var/log -v /Users/zj-db0972/k8s/learning/fluentd:/fluentd/etc -e FLUENTD_CONF=fluentd.conf fluent/fluentd/cgl-test:v0.0.1-1  
    * -v /var/log:/var/log 把宿主机的/var/log挂载到容器的/var/log目录下
    * -v /Users/zj-db0972/k8s/learning/fluentd:/fluentd/etc 宿主机fluentd.conf目录挂载到容器
   
   fluentd命令选项如下
   ```
   Usage: fluentd [options]
    -s, --setup [DIR=/etc/fluent]    install sample configuration file to the directory
    -c, --config PATH                config file path (default: /etc/fluent/fluent.conf)
        --dry-run                    Check fluentd setup is correct or not
        --show-plugin-config=PLUGIN  Show PLUGIN configuration and exit(ex: input:dummy)
    -p, --plugin DIR                 add plugin directory
    -I PATH                          add library path
    -r NAME                          load library
    -d, --daemon PIDFILE             daemonize fluent process
        --under-supervisor           run fluent worker under supervisor (this option is NOT for users)
        --no-supervisor              run fluent worker without supervisor
        --workers NUM                specify the number of workers under supervisor
        --user USER                  change user
        --group GROUP                change group
    -o, --log PATH                   log file path
        --log-rotate-age AGE         generations to keep rotated log files
        --log-rotate-size BYTES      sets the byte size to rotate log files
        --log-event-verbose          enable log events during process startup/shutdown
    -i CONFIG_STRING,                inline config which is appended to the config file on-the-fly
        --inline-config
        --emit-error-log-interval SECONDS
                                     suppress interval seconds of emit error logs
        --suppress-repeated-stacktrace [VALUE]
                                     suppress repeated stacktrace
        --without-source             invoke a fluentd without input plugins
        --use-v1-config              Use v1 configuration format (default)
        --use-v0-config              Use v0 configuration format
    -v, --verbose                    increase verbose level (-v: debug, -vv: trace)
    -q, --quiet                      decrease verbose level (-q: warn, -qq: error)
        --suppress-config-dump       suppress config dumping when fluentd starts
    -g, --gemfile GEMFILE            Gemfile path
    -G, --gem-path GEM_INSTALL_PATH  Gemfile install path (default: $(dirname $gemfile)/vendor/bundle)
   ```
   
5. 查看容器日志，如果有以下输出说明Fluentd启动成功
   ```
   2019-02-21 12:15:01 +0000 [info]: parsing config file is succeeded path="/fluentd/etc/fluentd.conf"
   2019-02-21 12:15:01 +0000 [warn]: To prevent events traffic jam, you should specify 2 or more 'flush_thread_count'.
   2019-02-21 12:15:01 +0000 [warn]: 'pos_file PATH' parameter is not set to a 'tail' source.
   2019-02-21 12:15:01 +0000 [warn]: this parameter is highly recommended to save the position to resume tailing.
   2019-02-21 12:15:01 +0000 [info]: using configuration file: <ROOT>
     <source>
       @type tail
       port 42185
       path "/var/log/docker-ce.log"
       time_format %Y-%m-%dT%H:%M:%SZ
       tag "docker_ce_logs"
       <parse>
         @type "syslog"
         time_format "%Y-%m-%dT%H:%M:%SZ"
       </parse>
     </source>
     <source>
       @type forward
     </source>
     <match **>
       @type elasticsearch
       host "elasticsearch"
       logstash_format true
       <buffer>
         flush_interval 10s
       </buffer>
     </match>
   </ROOT>
   2019-02-21 12:15:01 +0000 [info]: starting fluentd-1.3.2 pid=8 ruby="2.5.2"
   2019-02-21 12:15:01 +0000 [info]: spawn command to main:  cmdline=["/usr/bin/ruby", "-Eascii-8bit:ascii-8bit",   "/usr/bin/fluentd", "-c", "/fluentd/etc/fluentd.conf", "-p", "/fluentd/plugins", "--under-supervisor"]
   2019-02-21 12:15:01 +0000 [info]: gem 'fluent-plugin-elasticsearch' version '3.2.3'
   2019-02-21 12:15:01 +0000 [info]: gem 'fluentd' version '1.3.2'
   2019-02-21 12:15:01 +0000 [info]: adding match pattern="**" type="elasticsearch"
   2019-02-21 12:15:01 +0000 [info]: #0 Detected ES 6.x: ES 7.x will only accept `_doc` in type_name.
   2019-02-21 12:15:01 +0000 [warn]: #0 To prevent events traffic jam, you should specify 2 or more 'flush_thread_count'.
   2019-02-21 12:15:01 +0000 [info]: adding source type="tail"
   2019-02-21 12:15:01 +0000 [warn]: #0 'pos_file PATH' parameter is not set to a 'tail' source.
   2019-02-21 12:15:01 +0000 [warn]: #0 this parameter is highly recommended to save the position to resume tailing.
   2019-02-21 12:15:01 +0000 [info]: adding source type="forward"
   2019-02-21 12:15:01 +0000 [warn]: parameter 'port' in <source>
   ```

# 三. Fluentd命令
`Fluentd`核心是fluentd.conf配置文件，配置文件内容核心为`命令`和`插件`两部分。

`fluentd.conf`配置文件是由各种命令块组成，每一种命令都是为了完成某一种处理，命令与命令之间还可以通过Pipeline的形式处理和分发数据。  

Fluentd的命令主要有以下6个
1. source: 配置数据输入源
2. match: 配置数据输出目的地
3. filter: 数据处理逻辑
4. system: 系统配置
5. label: 数据分组
6. include: 引入其他文件

最常见的方式是`source`配置数据输入源，通过`filter`做流式的数据处理，最后通过`match`进行匹配分发。`match`是配置数据输出目的地，一旦匹配上就不会再继续往下匹配。

详细的Fluentd命令介绍请参考 [Fluentd 命令介绍](https://chenguolin.github.io/2019/02/27/Fluentd-Fluentd%E5%91%BD%E4%BB%A4%E4%BB%8B%E7%BB%8D/)

# 四. Fluentd插件
`Fluentd`核心是fluentd.conf配置文件，配置文件内容核心为`命令`和`插件`两部分。

`fluentd.conf`配置文件是由各种命令块组成，大多数命令都需要配置对应的插件，Fluentd支持6种类型插件，社区提供了超过500+的插件 [List of All Plugins](https://www.fluentd.org/plugins/all)。  

Fluentd的插件类型如下
1. Input Plugins: 输入插件，用于读取日志，用在`source`命令
2. Parser Plugins: 解析插件，用于修改日志输入事件格式，用在`source`命令
3. Filter Plugins: 过滤插件，用于修改日志事件流，用在`filter`命令
4. Formatter Plugins: 格式化插件，用于修改日志输出事件格式，用在`Match`命令
5. Buffer Pluginds: 缓存插件，用于缓存输出日志，用在`match`命令
6. Output Plugins: 输出插件，用于输出日志，用在`match`命令

详细的Fluentd插件介绍请参考 [Fluentd 插件介绍](https://chenguolin.github.io/2019/03/01/Fluentd-Fluentd-%E6%8F%92%E4%BB%B6%E4%BB%8B%E7%BB%8D/)

`fluentd.conf`配置文件每个命令配置，都会实例化一个对应插件的对象，每个插件实例有自己的plugin id。


# 五. Fluentd数据类型
`Fluentd`核心是fluentd.conf配置文件，配置文件内容核心为`命令`和`插件`两部分。每个`命令`块都需要配置对应的插件，而每个插件有一序列的参数。

每个参数都有特定的数据类型，Fluentd支持的数据类型如下。
1. string: 参数被解析成string，可以不适用单、双引号，同时也支持使用单、双引号引起来。例如`buffer_type file`
2. integer: 参数被解析成integer。例如`num_threads 4`
3. float: 参数被解析成float。例如`interval 4.5`
4. size: 参数被解析成多少bytes，支持以下4种类型
    * <INTEGER>k or <INTEGER>K: 多少Kb。例如`max_send_limit_bytes 1024K`
    * <INTEGER>m or <INTEGER>M: 多少M。例如`max_send_limit_bytes 1024M`
    * <INTEGER>g or <INTEGER>G: 多少G。例如`max_send_limit_bytes 1024G`
    * <INTEGER>t or <INTEGER>T: 多少T。例如`max_send_limit_bytes 1024T`
5. time: 参数被解析成时间，支持以下4种类型
    * <INTEGER>s: 多少秒。例如`flush_interval 3s`
    * <INTEGER>m: 多少分。例如`flush_interval 3m`
    * <INTEGER>h: 多少小时。例如`flush_interval 3h`
    * <INTEGER>d: 多少天。例如`flush_interval 3d`
6. array: 参数被解析成数组，例如`key ["key1", "key2"]`
7. hash: 参数被解析成JSON对象，例如`params {"key1":"value1", "key2":"value2"}`

# 六. Fluentd采集流程
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-pipeline.png?raw=true)

实际生产环境中使用Fluentd采集日志流程如上图所示

1. 输入插件读取日志数据，每个日志事件包括time、tag和record
    * time: 日志事件采集时间
    * tag: 日志事件标签，用于日志路由
    * record: 实际日志内容，JSON对象
2. 日志事件根据Fluentd路由进行一系列操作和处理
3. buffer 本质上是一组`chunk`的集合，由两部分组成`map + queue`
    * map: 日志事件会先写入map的某一个chunk，文件名格式为"{{prefix}}.log.`b`xxxxx.log"，通过`b`来标识是map文件
        * 如果chunk超过buffer_chunk_limit会自动刷新到queue
        * 如果flush_interval时间到了也会自动刷新到queue
    * queue: queue 也包含多个chunk，queue里面的chunk无法更改，等待flush到实际存储媒介例如File、Kafka、ES等，文件名格式为"{{prefix}}.log.`q`xxxxx.log"，通过`q`来标识是queue文件
        * 如果queue满了，会报队列溢出错误
        * 如果queue中的chunk写存储媒介失败，则会通过指数退避算法进行重试
4. 整个buffer缓存的大小 = map + queue

`为什么要采用Buffer？`  
> 1). 很多时候日志生产者和消费者的速率是不相等的，很容易造成日志生产过快导致来不及消费。这就会造成日志数据堆积，久而久之可能会造成数据丢失。所以，为了解决这个问题我们采用Buffer缓存日志。  
> 2). 为了解耦日志的生产者和消费组，保证数据可靠性。  
  
`Fluentd内部实现原理`  
> 1). 默认情况下Fluentd日志事件由Input插件线程默认处理，假设我们的Pipeline是这样的`in_tail -> filter_grep -> out_stdout`，那这个Pipeline是由in_tail插件线程处理，filter_grep和out_stdout插件是没有自己的线程。  
> 2). 如果有Buffer插件，那Output插件会有自己的线程处理flush buffer，假设我们的Pipeline是这样的`in_tail -> filter_grep -> out_forward`，那这个Pipeline会有2个线程。in_tail插件线程处理日志事件写入buffer，out_forward插件会有自己的线程处理flush buffer。

