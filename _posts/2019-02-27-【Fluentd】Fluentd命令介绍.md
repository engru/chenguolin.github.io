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

# 二. Fluentd命令
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

## ① source
`source`是`Fluentd`一切数据的输入源，每个`source`都是一个输入源。
```
# Receive events from 24224/tcp
# This is used by log forwarding and the fluent-cat command
<source>
   @type forward
   port 24224
</source>

# http://this.host:9880/myapp.access?json={"event":"data"}
<source>
   @type http
   port 9880
</source>
```

`Fluentd`原生就支持`http`和`forward`两个输入插件，`http`插件接收HTTP 9880端口请求传入的消息，`forward`接收TCP 24224端口的报文。除了这2个之外最常见的是`tail`输入插件用于读取文件。

`source`命令获取输入数据后会向`Fluentd`路由引擎提交一个事件，每个事件由3个部分组成`tag`、`time`、`record`。
1. tag: 用于告诉Fluentd路由引擎匹配哪个match输出
2. time: 事件生成时间
3. record: 事件内容

例如上面配置`fluentd.conf`第二个source例子，请求`http://this.host:9880/myapp.access?json={"event":"data"}`，source会提交一个事件如下所示。
```
tag: myapp.access
time: current time
record: {"event":"data"}
```

## ② match
`match`命令用于配置数据输出的目的地，通过匹配事件的`tag`来执行指定的输出插件进行数据输出。  
`match`不仅可以用来处理输出，还可以对数据进行一些处理后重新提交当成一个新的事件重新走一遍处理流程，例如可以用`rewrite_tag_filter`插件重新打tag。

```
# http://this.host:9880/myapp.access?json={"event":"data"}
<source>
   @type http
   port 9880
</source>

# Match events tagged with "myapp.access" and
# store them to /var/log/fluent/access.%Y-%m-%d
# Of course, you can control how you partition your data
# with the time_slice_format option.
<match myapp.access>
   @type file
   path /var/log/fluent/access
</match>
```

上面这个例子`<match myapp.access>`表示所有tag为mysqpp.access的事件最终都会通过file输出插件，输出到文件/var/log/fluent/access。

match匹配支持以下4种模式
```
1. *: 匹配单个tag部分，例如a.* 可以匹配a.b但是不能匹配a或者a.b.c       
2. **: 匹配0个或多个tag部分，例如a.** 可以匹配a、a.b和a.b.c
3. {X,Y,Z}: 匹配X、Y或Z，其中X、Y和Z是匹配模式，可以和*和**模式组合使用。例如{a,b}匹配a和b，但不匹配c；例如a.{b,c}可以匹配a.b和a.c
4. a b: <match a b> 可以匹配a或b，只要有一个匹配上即可 
```

`注意: match是从上往下匹配的，数据流一旦匹配上某个match就不会再继续往下匹配了，所以<match **>这样的全匹配必须要放到最后面位置。`

## ③ filter
`filter`命令用于处理数据流，多个filter之间串联成Pipeline对数据进行流式处理，最终交给`match`进行输出。

```
# http://this.host:9880/myapp.access?json={"event":"data"}
<source>
   @type http
   port 9880
</source>

<filter myapp.access>
   @type record_transformer
   <record>
      host_param "#{Socket.gethostname}"
   </record>
</filter>

<match myapp.access>
   @type file
   path /var/log/fluent/access
</match>
```

上面这个例子，filter收到数据`{"event":"data"}`后经过`record_transformer`插件处理新加一个host_param字段，数据会变成`{"event":"data", "host_param":"webserver1"}`，最后通过`match`输出到文件/var/log/fluent/access。

## ④ system
`system`命令用于设置Fluentd相关的配置，这些配置可以在Fluentd启动的时候配置也可以在fluentd.conf里设置，包括以下几点

1. log_level
2. suppress_repeated_stacktrace
3. emit_error_log_interval
4. suppress_config_dump
5. without_source

```
<system>
   # equal to -qq option
   log_level error
   # equal to --without-source option
   without_source
   # ...
</system>
```

## ⑤ label
`label`命令用不对特定分组的数据进行处理，常见的用法是在`source`里面使用`@lable`参数对数据进行打标签，通过`label`命令对数据进行分组处理。

```
<source>
   ### 这个任务指定了 label 为 @SYSTEM
   ### 会被发送给 <label @SYSTEM>
   ### 而不会被发送给下面紧跟的 filter 和 match
   @type tail
   @label @SYSTEM
</source>
 
<label @SYSTEM>
   <filter var.log.middleware.**>
      @type grep
      # ...
   </filter>
    
   <match **>
      @type s3
      # ...
   </match>
</label>
```

## ⑥ include
`include`命令用于导入其它文件，常用于导入一些公共文件，`include`支持导入绝对路径、相对路径以及HTTP。

```
# absolute path
@include /path/to/config.conf
          
# if using a relative path, the directive will use
# the dirname of this config file to expand the path
@include extra.conf
# glob match pattern
@include config.d/*.conf

# http
@include http://example.com/fluent.conf
```
