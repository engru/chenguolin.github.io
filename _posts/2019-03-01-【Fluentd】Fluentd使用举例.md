---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Fluentd
---

# 一. Simple Input -> Filter -> Output
```
<source>
  @type forward
</source>

<filter app.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match app.**>
  @type file
  # ...
</match>
```
1. 输入: 从forward插件接收日志事件
2. 处理: tag匹配上`app.**`的日志事件都会经过record_transformer插件处理，每个record新增一个hostname字段
3. 输出: tag匹配上`app.**`的日志事件使用file插件写到文件上

```
<source>
  @type forward
</source>

<source>
  @type tail
  tag system.logs
  # ...
</source>

<filter app.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match {app.**,system.logs}>
  @type file
  # ...
</match>
```
1. 输入: 从forward和tail这两个插件接收日志事件，从tail插件读取的日志tag会被设置为system.logs
2. 处理: tag匹配上`app.**`的日志事件都会经过record_transformer插件处理，每个record新增一个hostname字段
3. 输出: tag匹配上`app.**`的日志事件使用file插件写到文件上

# 二. Input -> Filter -> Output with Label
```
<source>
  @type forward
</source>

<source>
  @type dstat
  @label @METRICS # dstat events are routed to <label @METRICS>
  # ...
</source>

<filter app.**>
  @type record_transformer
  <record>
    # ...
  </record>
</filter>

<match app.**>
  @type file
  # ...
</match>

<label @METRICS>
  <match **>
    @type elasticsearch
    # ...
  </match>
</label>
```
1. 输入: 从forward和dstat这两个插件接收日志事件，从dstat插件读取的日志会被打上label @METRICS。dstat插件所读取的日志事件只会被发送给label指定@METRICS所包含的任务，而不会被其他filter、match获取到。
2. 处理: tag匹配上`app.**`的日志事件都会经过record_transformer插件处理，每个record新增一个hostname字段。（不包括label @METRICS的日志）
3. 输出: 分为两部分
    * tag匹配上`app.**`的日志事件使用file插件写到文件上 （不包括label @METRICS的日志）
    * label为@METRICS的日志事件使用elasticsearch写到es

# 三. Re-route event by tag
```
<match worker.**>
  @type route
  remove_tag_prefix worker
  add_tag_prefix metrics.event

  <route **>
    copy # For fall-through. Without copy, routing is stopped here. 
  </route>
  <route **>
    copy
    @label @BACKUP
  </route>
</match>

<match metrics.event.**>
  @type stdout
</match>

<label @BACKUP>
  <match metrics.event.**>
    @type file
    path /var/log/fluent/bakcup
  </match>
</label>
```
1. tag匹配上`worker.**`的日志事件，tag前缀会被改成metrics.event，同时会拷贝一份设置label @BACKUP
2. tag匹配上`metrics.event.**`的日志事件使用stdout插件输出到标准输出
3. 所有label为@BACKUP的日志事件，如果tag匹配上`metrics.event.**`会使用file插件写到/var/log/fluent/bakcup这个文件

# 四. Re-route event by record content
```
<source>
  @type forward
</source>

# event example: app.logs {"message":"[info]: ..."}
<match app.**>
  @type rewrite_tag_filter
  rewriterule1 message ^\[(\w+)\] $1.${tag}
</match>

# send mail when receives alert level logs
<match alert.app.**>
  @type mail
  # ...
</match>

# other logs are stored into file
<match *.app.**>
  @type file
  # ...
</match>
```
1. 输入: 从forward这个插件接收日志事件
2. 输出
   * tag匹配上`app.**`的日志事件，tag会被改写
   * tag匹配上`alert.app.**`的日志事件使用mail插件处理
   * tag匹配上`*.app.**`的日志事件使用file插件写入文件

# 五. Re-route event to other Label
```
<source>
  @type forward
</source>

<match app.**>
  @type copy
  <store>
    @type forward
    # ...
  </store>
  <store>
    @type relabel
    @label @NOTIFICATION
  </store>
</match>

<label @NOTIFICATION>
  <filter app.**>
    @type grep
    regexp1 message ERROR
  </filter>

  <match app.**>
    @type mail
  </match>
</label>
```
1. 输入: 从forward这个插件接收日志事件
2. 输出:
   * tag匹配上`app.**`的日志事件，会被拷贝2份，其中一份会被label会被设置为@NOTIFICATION
   * label为@NOTIFICATION的日志事件会先经过grep插件处理，再使用mail插件处理

