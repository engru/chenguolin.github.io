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

RUN echo "10.10.0.14 cgl.test.com" >> /etc/hosts
```

问题的根本原因是 hosts 文件其实并不是存储在Docker镜像中的，/etc/hosts, /etc/resolv.conf 和 /etc/hostname，是存在主机上的 /var/lib/docker/containers/{docker_id} 目录下，容器启动时是通过 mount 将这些文件挂载到容器内部的。因此如果在容器中修改这些文件，修改部分不会存在于容器的可读写层，而是直接写入这3个文件中。容器重启后修改内容不存在的原因是Docker每次创建新容器时，会根据当前 docker0 下的所有节点的IP信息重新建立hosts文件。也就是说，你的修改会被Docker给自动覆盖掉。

# 二. 解决方案
## ① docker run 添加 --add-host 参数
我们在运行容器的时候添加 --add-host 参数，容器运行命令为 `docker run -it --add-host=cgl.test.com:10.10.0.14 cgl-test-hosts:v0.0.1`,查看 hosts 是否有设置成功，发现容器内 /etc/hosts 已经有对应的域名解析配置了。

```
/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
10.10.0.14	cgl.test.com        //设置成功了
172.17.0.2	243fa854d215
```

注意: 虽然docker build命令也提供了 --add-host 选项，但是其实并不会生效。具体的原因可以参考 `the --add-host feature during build is designed to allow overriding a host during build, but not to persist that configuration in the image.`

## ② 进入容器修改
直接进入容器修改是最简单的方式，缺点是修改完之后每次重新启动一个新的容器都需要重新改动，不是很方便。

```
$ docker run -it cgl-test-hosts:v0.0.1
/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	21ab34d0fe34

/ # echo "10.10.0.14 cgl.test.com" >> /etc/hosts         //写入容器内/etc/hosts文件

/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	21ab34d0fe34
10.10.0.14 cgl.test.com      //设置成功了
```

## ③ 通过脚本注入镜像
通过前面的介绍我们知道在Dockerfile内直接修改 /etc/hosts 文件是无效的，因此我们需要把配置放在一个shell脚本内，通过Dockerfile把脚本注入镜像，同时设置容器的启动的命令为该shell脚本。

Dockerfile 内容如下
```
FROM alpine:3.5

COPY modify_hosts.sh /www/cgl/modify_hosts.sh
RUN chmod 775 /www/cgl/run.sh

# run the script that starts container
ENTRYPOINT /www/cgl/run.sh
```

run.sh 脚本内容如下
```
#!/bin/sh

## set hosts
echo "10.10.0.14 cgl.test.com" >> /etc/hosts

## start application
sleep 36000
```

1. 镜像构建: docker build -t cgl-test-hosts:v0.0.2 .
2. 运行容器并查看是否设置成功
```
$ docker run -itd cgl-test-hosts:v0.0.2
  aadb29a5716dd97d3a042808dbe391b5337e6b87205f4b3db670c640776414cf
$ docker exec -it aadb29a5716dd97d3a042808dbe391b5337e6b87205f4b3db670c640776414cf /bin/sh
/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	aadb29a5716d
10.10.0.14 cgl.test.com       //设置成功了
```

