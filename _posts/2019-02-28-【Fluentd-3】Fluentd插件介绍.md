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

# 二. Fluentd插件
`Fluentd`核心是fluentd.conf配置文件，配置文件内容核心为`命令`和`插件`两部分。

`fluentd.conf`配置文件是由各种命令块组成，大多数命令都需要配置对应的插件，Fluentd支持6种类型插件，社区提供了超过500+的插件 [List of All Plugins](https://www.fluentd.org/plugins/all)。  

Fluentd的插件类型如下
1. Input Plugins: 输入插件，用于读取日志，用在`source`命令
2. Parser Plugins: 解析插件，用于修改日志输入事件格式，用在`source`命令
3. Filter Plugins: 过滤插件，用于修改日志事件流，用在`filter`命令
4. Formatter Plugins: 格式化插件，用于修改日志输出事件格式，用在`Match`命令
5. Buffer Pluginds: 缓存插件，用于缓存输出日志，用在`match`命令
6. Output Plugins: 输出插件，用于输出日志，用在`match`命令

插件的通用参数如下
1. @type: 标识插件类型
2. @id: 插件id
3. @label: 给日志事件打label
4. @log_level: 给每个插件设置log level

## ① Input Plugins
Input插件用于读取日志事件，Input插件通常会创建一个线程socket和一个监听socket。  
`Input插件用在source命令`

常用的Input插件如下所示

1. in_tail
   ```
   <source>
      @type tail
      path /var/log/httpd-access.log
      pos_file /var/log/td-agent/httpd-access.log.pos
      tag apache.access
      <parse>
         @type apache2
      </parse>
   </source>
   ```
   
   in_tail插件从文件读取日志，默认从文件尾开始读取，也可以设置从文件头开始读取。
   
   重要的参数  
   1). @type: 值为tail表示使用in_tail这个插件  
   2). tag: 日志事件tag   
   3). path: 读取的文件路径，多个路径使用","分割  
   4). exclude_path: 剔除哪些文件路径，例如exclude_path ["/path/to/*.gz", "/path/to/*.zip”]  
   5). refresh_interval: 读取路径刷新间隔，默认60s  
   6). pos_file: 记录最后读取日志位置文件  
   7). read_from_head 是否从文件头开始读取，默认为false。如果path设置的是一个动态路径例如/path/*/current，需要设置read_from_head为true

2. in_forward
   ```
   <source>
      @type forward
      port 24224
      bind 0.0.0.0
   </source>
   ```
   
   in_forward插件用于接收监听TCP报文，也支持接收UDP心跳消息，主要用于从其它Fluentd实例接收日志事件。
   
   重要参数  
   1). @type: 值为forward表示使用in_forward这个插件  
   2). port: 监听的端口，默认为24224   
   3). bind: 监听的地址 0.0.0.0表示监听所有地址  
   4). source_host_key: 客户端hostname

3. in_udp
   ```
   <source>
      @type udp
      tag mytag # required
      <parse>
         @type regexp
         expression /^(?<field1>\d+):(?<field2>\w+)$/
      </parse>
      port 20001               # optional. 5160 by default
      bind 0.0.0.0             # optional. 0.0.0.0 by default
      message_length_limit 1MB # optional. 4096 bytes by default
   </source>
   ```
   
   in_udp插件用于接收UDP数据包。
   
   重要参数  
   1). @type: 值为udp表示使用in_udp这个插件  
   2). tag: 日志事件tag  
   3). port: 监听的端口，默认为5160  
   4). bind: 监听的地址 0.0.0.0表示监听所有地址  
   5). source_host_key: 客户端hostname  
   6). message_length_limit: 消息最大字节数  
   7). parse: 解析udp数据包  

4. in_tcp
   ```
   <source>
      @type tcp
      tag tcp.events # required
      <parse>
         @type regexp
         expression /^(?<field1>\d+):(?<field2>\w+)$/
      </parse>
      port 20001   # optional. 5170 by default
      bind 0.0.0.0 # optional. 0.0.0.0 by default
      delimiter \n # optional. \n (newline) by default
   </source>
   ```
   
   in_tcp插件用于接收tcp数据包，一般使用in_forward插件而不使用in_tcp插件。
   
   重要参数  
   1). @type: 值为tcp表示使用in_tcp这个插件  
   2). tag: 日志事件的tag  
   3). port: 监听的端口，默认为5160  
   4). bind: 监听的地址 0.0.0.0表示监听所有地址  
   5). source_host_key: 客户端hostname  
   6). parse: 解析tcp数据包  

5. in_http
   ```
   <source>
      @type http
      port 9880
      bind 0.0.0.0
      body_size_limit 32m
      keepalive_timeout 10s
   </source>
   ```
   
   in_http插件通过HTTP请求来接收日志。
   
   重要参数  
   1). @type: 值为http表示使用in_http这个插件  
   2). port: 监听的端口，默认为5160  
   3). bind: 监听的地址 0.0.0.0表示监听所有地址  
   4). body_size_limit: post数据大小限制  
   5). keepalive_timeout: 连接keep alive时间  
  
6. in_syslog
   ```
   <source>
       @type syslog
       port 5140
       bind 0.0.0.0
       tag system
   </source>
   ```
   
   in_syslog插件从syslog读取日志。
   
   重要参数  
   1). @type 值为syslog表示使用in_syslog这个插件  
   2). tag: 日志事件的tag  
   3). port: 监听的端口，默认为5140  
   4). bind: 监听的地址 0.0.0.0表示监听所有地址  

7. in_unix
   ```
   <source>
      @type unix
      path /path/to/socket.sock
   </source>
   ```
   
   in_unix插件支持从unix socket接收日志。
   
   重要参数  
   1). @type: 值为unix表示使用in_unix这个插件  
   2). path: socket文件路径

8. in_dummy
   ```
   <source>
      @type dummy
      dummy {"hello":"world"}
   </source>
   ```
   
   in_dummy插件用来生成假数据用于测试和debug。
   
   重要参数  
   1). @type: 值为dummy表示使用in_dummy这个插件  
   2). tag: 日志事件的tag  
   3). size: 每个发出的Event的个数，默认为1  

9. in_multiprocess
   ```
   <source>
      @type multiprocess
      <process>
          cmdline -c /etc/fluent/fluentd_child1.conf --log /var/log/fluent/fluentd_child1.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
      </process>
      <process>
          cmdline -c /etc/fluent/fluentd_child2.conf --log /var/log/fluent/fluentd_child2.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
      </process>
      <process>
          cmdline -c /etc/fluent/fluentd_child3.conf --log /var/log/fluent/fluentd_child3.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
      </process>
   </source> 
   ```
   
   in_multiprocess 插件支持Fluentd使用多核CPU运行多个子进程，使用这个插件可以提高Fluentd的处理能力。

   安装in_multiprocess插件  
   $ fluent-gem install fluent-plugin-multiprocess

   重要参数  
   1). @type: multiprocess 表示in_multiprocessor插件  
   2). process: 子进程配置  
   3). cmdline: 子进程启动命令  
   4). sleep_before_start: 等待多久启动子进程  
   5). sleep_before_shutdown: 等待多久关闭子进程  
    

## ② Parser Plugins
Parser插件，用于修改日志输入事件格式。有时输入插件的format参数无法解析用户的自定义数据格式，所以用户需要自定义Parser插件。  
`Parser插件用在source命令，format字段。`

常用的Parser插件如下所示

1. regexp
   ```
   <source>
     @type tail
     path /path/to/input/file
     format /^\[(?<logtime>[^\]]*)\] (?<name>[^ ]*) (?<title>[^ ]*) (?<id>\d*)$/
     time_key logtime
     time_format %Y-%m-%d %H:%M:%S %z
     types id:integer
   </source>
   ```
   
   regexp插件通过正则表达式解析日志，如果format参数值开始和结束都是`/`说明是一个正则表达式。
   
   举例
   ```
   日志事件: [2013-02-28 12:00:00 +0900] alice engineer 1
   输出结果
   
   time:
   1362020400 (22013-02-28 12:00:00 +0900)

   record:
   {
     "name" : "alice",
     "title": "engineer",
     "id"   : 1
   }
   ```
   
   重要参数  
   1). time_key: 日志事件时间字段，默认为time  
   2). time_format: 日志事件时间格式  
   3). keep_time_key: 是否在record中保留原有时间字段，默认为false  
   4). types: 字段类型定义，默认情况下都认为是string类型  

2. apache2
   ```
   <source>
     @type tail
     path /path/to/input/file
     format /^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$/
     time_format %Y-%m-%d %H:%M:%S %z
   </source>
   ```
   
   apache2插件用于解析apache2日志。
   
   举例
   ```
   日志事件: 192.168.0.1 - - [28/Feb/2013:12:00:00 +0900] "GET / HTTP/1.1" 200 777 "-" "Opera/12.0"
   输出结果
   
   time:
   1362020400 (28/Feb/2013:12:00:00 +0900)
   
   record:
   {
     "user"   : nil,
     "method" : "GET",
     "code"   : 200,
     "size"   : 777,
     "host"   : "192.168.0.1",
     "path"   : "/",
     "referer": nil,
     "agent"  : "Opera/12.0"
   }
   ```
   
   重要参数  
   1). keep_time_key: 是否在record中保留原有时间字段，默认为false  

3. nginx
   ```
   <source>
     @type tail
     path /path/to/input/file
     format /^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$/
     time_format %d/%b/%Y:%H:%M:%S %z
   </source>
   ```
   
   nginx插件用于解析nginx日志。
      
   举例
   ```
   日志事件: 127.0.0.1 192.168.0.1 - [28/Feb/2013:12:00:00 +0900] "GET / HTTP/1.1" 200 777 "-" "Opera/12.0"
   输出结果
   
   time:
   1362020400 (28/Feb/2013:12:00:00 +0900)
   
   record:
   {
     "remote" : "127.0.0.1",
     "host"   : "192.168.0.1",
     "user"   : "-",
     "method" : "GET",
     "path"   : "/",
     "code"   : "200",
     "size"   : "777",
     "referer": "-",
     "agent"  : "Opera/12.0"
   }
   ```
   
   重要参数  
   1). keep_time_key: 是否在record中保留原有时间字段，默认为false  
   2). types: 字段类型定义，默认情况下都认为是string类型  
   
4. json
   ```
   <source>
     @type tail
     path /path/to/input/file
     format json
     time_format %d/%b/%Y:%H:%M:%S %z
   </source>
   ```
   
   json插件用于解析json格式日志。
      
   举例
   ```
   日志事件: {"time":1362020400,"host":"192.168.0.1","size":777,"method":"PUT"}
   输出结果
   
   time:
   1362020400 (2013-02-28 12:00:00 +0900)
   
   record:
   {
     "host"  : "192.168.0.1",
     "size"  : 777,
     "method": "PUT",
   }
   ```
   
   重要参数  
   1). time_key: 日志事件时间字段，默认为time  
   2). time_format: 日志事件时间格式  
   3). keep_time_key: 是否在record中保留原有时间字段，默认为false  

5. none
   ```
   <source>
     @type tail
     path /path/to/input/file
     format none
   </source>
   ```
   
   none插件用于把一行日志解析为单个字段。
      
   举例
   ```
   日志事件: Hello world. I am a line of log!
   输出结果
   
   time:
   1362020400 (current time)
   
   record:
   {"message":"Hello world. I am a line of log!"}
   ```
   
   重要参数  
   1). message_key: 单行日志字段，默认为message

## ③ Filter Plugins
Filter插件，用于修改日志事件record，包括以下几个方面  
1). 通过一个或多个字段进行日志过滤  
2). 给每个日志事件添加新字段  
3). 日志事件删除某些字段  

`Parser插件用在filter命令`

常用的Parser插件如下所示

1. record_transformer
   ```
   <filter foo.bar>
     @type record_transformer
     <record>
       hostname "#{Socket.gethostname}"
       tag ${tag}
     </record>
   </filter>
   ```
   
   record_transformer插件用于更新日志事件record
   
   举例
   ```
   日志事件record: {"message":"hello world!"}

   输出结果
   {"message":"hello world!", "hostname":"db001.internal.example.com", "tag":"foo.bar"}
   ```
   
   重要参数  
   1). record: 新增字段配置  
   2). keep_keys: 保留哪些keys，需要设置renew_record为true  
   3). remove_keys: 删除哪些keys  
   
2. grep
   ```
   <filter foo.bar>
     @type grep
     <regexp>
       key message
       pattern cool
     </regexp>
     <regexp>
       key hostname
       pattern ^web\d+\.example\.com$
     </regexp>
     <exclude>
       key message
       pattern uncool
     </exclude>
   </filter>
   ```
   
   grep插件用于过滤特定的日志事件
   
   举例
   ```
   上面插件配置的意思是
   1. 日志事件record message字段包含cool
   2. 日志事件record hostname字段需要符合正则表达式 ^web\d+\.example\.com$
   3. 日志事件record message字段不包含uncool
   ```

   重要参数  
   1). regexp: 包含规则配置，key是字段名称，value可以是值也可以是正则表达式  
   2). regexp: 不包含规则配置，key是字段名称，value可以是值也可以是正则表达式  

3. stdout
   ```
   <filter pattern>
     @type stdout
   </filter>
   ```
   
   stdout插件打印日志事件到标准输出，用于测试和debug
   
   示例
   ```
   原始日志事件
   time: 2015-05-02 12:12:17 +0900
   tag: stdout_tag
   record: {"field1":"value1","field2":"value2"}
   
   输出结果
   2015-05-02 12:12:17 +0900 tag: {"field1":"value1","field2":"value2"}
   由三部分组成 time tag: record
   ```


## ④ Formatter Plugins
Formatter插件，用于修改日志输出事件格式。有时输出插件的format参数无法解析用户的自定义数据格式，所以用户需要自定义Formatter插件。  
`Formatter插件用在match命令，format字段。`

常用的Formatter插件如下所示

1. out_file
   ```
   <match **>
      @type file
      format out_file
   </match>
   ```
   
   out_file插件用于输出time、tag和record 使用分隔符分割
   
   示例
   ```
   原始日志事件
   tag:    app.event
   time:   1362020400t
   record: {"host":"192.168.0.1","size":777,"method":"PUT"}

   输出结果
   2013-02-28T12:00:00+09:00\tapp.event\t{"host":"192.168.0.1","size":777,"method":"PUT"}
   ```
   
   重要参数  
   1). delimiter: time、tag和record字段分隔符，默认为\t  
   2). output_tag: 是否输出tag字段，默认为true  
   3). output_time: 是否输出time字段，默认为true  
   
2. json
   ```
   <match **>
      @type file
      format json
      include_tag_key true
      tag_key event_tag
   </match>
   ```
   
   json插件用于把time、tag和record等字段输出成json格式
   
   示例
   ```
   原始日志事件:
   tag:    app.event
   time:   1362020400
   record: {"host":"192.168.0.1","size":777,"method":"PUT"}
   
   输出结果
   {"host":"192.168.0.1","size":777,"method":"PUT","event_tag":"app.event"}
   ```
   
   重要参数  
   1). add_newline: 是否添加换行符\n，默认为true  
   2). include_time_key: 输出json是否包含time字段，默认为false  
   3). time_key: 输出的time字段名称  
   4). include_tag_key: 输出json是否包含tag字段，默认为false  
   5). tag_key: 输出的tag的字段名称
   
3. single_value
   ```
   <match **>
      @type file
      format single_value
      message_key message
   </match>
   ```
   
   single_value插件用于把record中的message输出成单行文本。
   
   示例
   ```
   原始日志事件
   tag:    app.event
   time:   1362020400
   record: {"message":"Hello from Fluentd!"}
   
   输出结果
   Hello from Fluentd!\n
   ```
   
   重要参数  
   1). message_key: record日志消息字段，默认为message

## ⑤ Buffer Pluginds
Buffer插件用于缓存输出的日志事件，被输出插件使用。`buffer`本质上是一组`chunk`的集合，一个`chunk`包含多个日志事件。  
`Buffer插件用在match命令，buffer_tyep字段。详细使用可以参考最后一章【Fluentd采集流程】`

常用的Buffer插件如下所示

1. memory
   ```
   <match pattern>
     @type file
     buffer_type memory
     buffer_chunk_limit 64M
     buffer_queue_limit 512
     flush_interval 3s
     flush_at_shutdown true
     retry_wait 1s
   </match>
   ```

   memory插件表示使用内存缓存chunk，如果Fluentd进程停止的时候来不及输出的chunk将被删除。
   
   重要参数  
   1). buffer_type: 缓存类型  
   2). buffer_chunk_limit: 每个buffer chunk的大小限制，默认为8M  
   3). buffer_queue_limit: buffer queue长度限制，默认是64个chunk  
   4). flush_interval: chunk刷新到queue的间隔  
   5). flush_at_shutdown: Fluentd进程关闭的时候是否flush，默认为true  
   6). retry_wait: flush重试的种子数，重试采用指数退避算法  

2. file
   ```
   <match pattern>
     @type file
     buffer_type file
     buffer_path /var/log/td-agent/buffer/td
     buffer_chunk_limit 64M
     buffer_queue_limit 512
     flush_interval 3s
     flush_at_shutdown true
     retry_wait 1s
   </match>
   ```
   
   file插件表示使用文件缓存chunk，在磁盘上存储实际chunk数据。`这里的文件只能使用本地文件系统，而不能使用远程文件系统例如HDFS、NFS等等`
   
   重要参数  
   1). buffer_type: 缓存类型  
   2). buffer_chunk_limit: 每个buffer chunk的大小限制，默认为8M  
   3). buffer_queue_limit: buffer queue长度限制，默认是64个chunk   
   4). flush_interval: chunk刷新到queue的间隔  
   5). flush_at_shutdown: Fluentd进程关闭的时候是否flush，默认为true  
   6). retry_wait: flush重试的种子数，重试采用指数退避算法  


## ⑥ Output Plugins
Output插件用于输出日志事件，可分为三种类型非缓存插件、缓存插件和时间切片插件。  
`Output插件用在match命令`

1. 非缓存插件: 非缓存输出插件不会缓存数据，会立即输出日志事件
2. 缓存插件: 缓存输出插件维护一个chunk buffer和chunk queue，通过chunk limit和queue limit可以来控制缓存刷新
3. 事件切片插件: 事件切片输出插件实际上是一种Bufferred插件，但是这些chunk是按时间写入

常用的Output插件如下所示

1. file
   ```
   <match pattern>
     @type file
     path /var/log/fluent/myapp
     time_format %Y%m%dT%H%M%S%z
     compress gzip
     utc
   </match>
   ```
   
   file插件用于输出日志到文件
   
   重要参数  
   1). @type: file表示使用的out_file插件   
   2). path: 输出文件路径  
   3). append: chunk是否添加到已经存在的文件，默认情况是每个chunk生成一个文件  
   4). format: 文件内容格式，默认是out_file（即Formatter Plugins的out_file插件）  
   5). time_format: 时间格式  
   6). compress: 是否使用gzip压缩  
   7). utc: 是否使用UTC时间  
   
2. s3
   ```
   <match pattern>
       @type s3
       aws_key_id YOUR_AWS_KEY_ID
       aws_sec_key YOUR_AWS_SECRET_KEY
       s3_bucket YOUR_S3_BUCKET_NAME
       s3_region ap-northeast-1
       path logs/
      
       # if you want to use ${tag} or %Y/%m/%d/ like syntax in path / s3_object_key_format,
       # need to specify tag for ${tag} and time for %Y/%m/%d in <buffer> argument.
       <buffer tag,time>
           @type file
           path /var/log/fluent/s3
           timekey 3600 # 1 hour partition
           timekey_wait 10m
           timekey_use_utc true # use utc
           chunk_limit_size 256m
       </buffer>
   </match>
   ```
   
   s3插件把日志写入aws S3服务。

   重要参数  
   1). @type: 值为s3表示out_s3插件  
   2). aws_key_id: aws access key id  
   3). aws_sec_key: aws secret key  
   4). path: s3路径前缀，默认为空表示没有前缀  
   5). buffer: 日志buffer配置  

3. kafka
   ```
   <match pattern>
      @type kafka_buffered
      # list of seed brokers
      brokers <broker1_host>:<broker1_port>,<broker2_host>:<broker2_port>
      # buffer settings
      buffer_type file
      buffer_path /var/log/td-agent/buffer/td
      flush_interval 3s
      # topic settings
      default_topic messages
      # data type settings
      output_data_type json
      compression_codec gzip
      # producer settings
      max_send_retries 1
      required_acks -1
   </match>
   ```
   
   kafka_buffered插件把日志输出到kafka
   
   安装kafka_buffered插件  
   $ fluent-gem install fluent-plugin-kafka
    
   重要参数  
   1). @type: 值为kafak_buffered表示使用kafka_buffered这个插件  
   2). brokers kafka broker配置  
   3). default_topic 默认topic名称  
   4). out_data_type 输出数据类型 json、ltsv、msgpack，msgpack数据紧凑速度最快   
   5). max_send_retries 发消息到broker最大重试次数  
   6). required_acks 是否需要没个写数据都需要ack确认  
   7). buffer参数  
       buffer_type 缓存类型file或者memory  
       buffer_path 如果buffer_type使用file 则需要设置buffer_path  
       buffer_chunk_limit 每个chunk块大小限制  
       buffer_queue_limit 每个queue chunk数量限制  
       flush_internal buffer刷新到queue的时间间隔  
   
4. Elasticsearch
   ```
   <match my.logs>
      @type elasticsearch
      host localhost
      port 9200
      logstash_format true
   </match>
   ```
   
   elasticsearch插件把日志写入elasticsearch      
      
   安装out_elasticsearch插件  
   $ gem install fluent-plugin-elasticsearch
     
   重要参数  
   1). @type: 值为elasticsearch 表示out_elasticsearch插件  
   2). host: es节点host，默认为localhost  
   3). port: es节点port，默认为9200  
   4). hosts: 多个es节点，例如host1:port1,host2:port2,host3:port3  
   5). user: es节点登录用户名  
   6). password: es节点登录密码  
   7). index_name: es index名称，默认为fluentd  
   8). logstash_format: 设置es index名称为logstash-%Y.%m.%d，替代@index_name参数  

5. copy
   ```
   <match myevent.file_and_elasticsearch>
      @type copy
      <store>
         @type file
         path /var/log/fluent/myapp
         compress gzip
         <format>
            localtime false
         </format>
         <buffer time>
            timekey_wait 10m
            timekey 86400
            timekey_use_utc true
            path /var/log/fluent/myapp
         </buffer>
         <inject>
            time_format %Y%m%dT%H%M%S%z
            localtime false
         </inject>
         </store>
      <store>
         @type elasticsearch
         host fluentd
         port 9200
         index_name fluentd
         type_name fluentd
      </store>
   </match>
   ```
   
   copy插件拷贝日志到多个输出
   
   重要参数  
   1). @type 职位copy表示使用out_copy插件  
   2). deep_copy: copy插件默认所有插件共享日志Record，配置了deep_copy可以做深拷贝  
   
6. roundrobin
   ```
   <match pattern>
       @type roundrobin
       <store>
          @type tcp
          host 192.168.1.21
          weight 3
       </store>
       <store>
          @type tcp
          host 192.168.1.22
          weight 2
       </store>
       <store>
          @type tcp
          host 192.168.1.23
          weight 1
       </store>
   </match>
   ```
   
   roundrobin插件把日志通过加权轮询算法输出到多个输出

7. stdout
   ```
   <match pattern>
      @type stdout
   </match>
   ```
   
   stdout插件把日志输出到stdout

8. null
   ```
   <match pattern>
      @type null
   </match>
   ```
   
   null插件用于丢掉日志

9. mongo
   ```
   <match mongo.**>
      @type mongo
      host fluentd
      port 27017
      database fluentd
      collection test
      # for capped collection
      capped
      capped_size 1024m
      # authentication
      user michael
      password jordan
      <inject>
        # key name of timestamp
        time_key time
      </inject>
      <buffer>
        # flush
        flush_interval 10s
      </buffer>
   </match>
   ```
   
   mongo 插件写日志到MongoDB
     
   安装插件  
   $ gem install fluent-plugin-mongo
 
   重要参数  
   1). @type: 值为mongo表示使用out_mongo这个插件  
   2). host: MongoDB host  
   3). port: MongoDB port  
   4). database: MongoDB 数据库  
   
10. relabel
    ```
    <match pattern>
       @type relabel
       @label @foo
    </match>
      
    <label @foo>
       <match pattern>
         ...
       </match>
    </label>
    ```
    
    relabel插件relabel日志tag
     
    重要参数  
    1). @type: 值为relabel表示out_relabel插件  
    2). label: 表示label值  

11. rewrite_tag_filter
    ```
    <match app.component>
       @type rewrite_tag_filter
       <rule>
          key message
          pattern /^\[(\w+)\]/
          tag $1.${tag}
       </rule>
    </match>
    ```
        
    rewrite_tag_filter插件用于重写日志tag，日志将会从新
    
    举例
    ```
    +------------------------------------------+        +------------------------------------------------+
    | original record                                   |        | rewritten tag record                  |
    |------------------------------------------|        |------------------------------------------------|
    | app.component {"message":"[info]: ..."}  | +----> | info.app.component {"message":"[info]: ..."}   |
    | app.component {"message":"[warn]: ..."}  | +----> | warn.app.component {"message":"[warn]: ..."}   |
    | app.component {"message":"[crit]: ..."}  | +----> | crit.app.component {"message":"[crit]: ..."}   |
    | app.component {"message":"[alert]: ..."} | +----> | alert.app.component {"message":"[alert]: ..."} |
    +------------------------------------------+        +------------------------------------------------+
    ```
    
    安装out_rewrite_tag_filter插件  
    $ gem install fluent-plugin-rewrite-tag-filter
   
12. webhdfs
    ```
    <match access.**>
       @type webhdfs
       host namenode.your.cluster.local
       port 50070
       path "/path/on/hdfs/access.log.%Y%m%d_%H.#{Socket.gethostname}.log"
       <buffer>
         flush_interval 10s
       </buffer>
    </match>
    ```
    
    webhdfs插件把日志写入HDFS文件系统
     
    安装插件  
    $ gem install fluent-plugin-webhdfs
        
    重要参数  
    1). @type 值为webhdfs表示out_webhdfs插件  
    2). host: 表示namenode host  
    3). port: 表示namenode port  
    4). path: 表示HDFS文件路径  

    
