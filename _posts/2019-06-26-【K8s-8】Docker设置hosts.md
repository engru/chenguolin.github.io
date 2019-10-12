---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - K8s
---

# 一. 背景
很多时候我们会需要测试某个域名是否能够正常访问，如果域名解析还未生效我们一般需要先通过设置 hosts 的方式来保证能够正常访问。但是在Docker容器环境下，设置hosts和传统的物理机会有些区别。

正常情况下我们会想到在Dockerfile里面设置/etc/hosts，Dockerfile内容如下所示。使用Dockerfile完成镜像构建之后，运行容器之后发现容器内/etc/hosts文件并没有 10.255.0.14 cgl.test.com 这行配置。
```
FROM alpine:3.5

RUN echo "10.255.0.14 cgl.test.com" >> /etc/hosts
```

问题的根本原因是 hosts 文件其实并不是存储在Docker镜像中的，/etc/hosts, /etc/resolv.conf 和 /etc/hostname，是存在主机上的 /var/lib/docker/containers/{docker_id} 目录下，容器启动时是通过 mount 将这些文件挂载到容器内部的。因此如果在容器中修改这些文件，修改部分不会存在于容器的可读写层，而是直接写入这3个文件中。容器重启后修改内容不存在的原因是Docker每次创建新容器时，会根据当前 docker0 下的所有节点的IP信息重新建立hosts文件。也就是说，你的修改会被Docker给自动覆盖掉。

# 二. 解决方案
