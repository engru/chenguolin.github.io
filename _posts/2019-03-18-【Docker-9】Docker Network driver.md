---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. 概述
TODO (@玄苦)

参考文章  
1. https://mp.weixin.qq.com/s/Lyy69rB3k9l30nEfFo_Wrg
2. https://mp.weixin.qq.com/s/NHt6lYzcT0-ZLwstXWoV8g
3. https://mp.weixin.qq.com/s/C05a5TrZs5_IdJK4ugiUbg
4. https://www.infoq.cn/article/docker-kernel-knowledge-namespace-resource-isolation
5. https://www.infoq.cn/article/docker-container-management-libcontainer-depth-analysis
6. https://www.infoq.cn/article/docker-standard-container-execution-engine-runc

# 二. Docker net
docker run 创建Docker容器时，可以用`--net`选项指定容器的网络模式，`--net`参数允许容器加入另一个容器的Network Namespace中，如果指定`--net=host`表示这个容器进程不会设置 Network Namespace，和宿主机共享网络栈。

Docker有以下4种网络模式
1. `host`: `--net=host`指定
2. `container`: ，`--net=container:NAMEorID`指定
3. `none`: `--net=none`指定
4. `bridge`: `--net=bridge`指定 (Docker默认设置)

## ① host模式
如果启动容器的时候使用 `host` 模式，那么这个容器将不会获得一个独立的 Network Namespace，而是和宿主机共用一个 Network Namespace。容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

例如我们在 `10.10.101.105` 的机器上用 host 模式启动一个含有 web 应用的 Docker 容器，监听 tcp 80 端口。当我们在容器中执行任何类似 ifconfig 命令查看网络环境时，看到的都是宿主机上的信息。而外界访问容器中的应用，则直接使用`10.10.101.105:80`即可，不用任何 NAT 转换，就如直接跑在宿主机中一样。

但是，容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

`当使用host模式网络时，容器实际上继承了宿主机的IP地址。该模式比bridge模式更快（因为没有路由开销），但是它将容器直接暴露在公共网络中，是有安全隐患的。`

![](http://dockone.io/uploads/article/20160503/5d7564e6eb8554412bd74e6772a336b4.png)

```
$ docker run -d --net=host ubuntu:14.04 tail -f /dev/null
$ ip addr | grep -A 2 eth0:
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
link/ether 06:58:2b:07:d5:f3 brd ff:ff:ff:ff:ff:ff
inet **10.0.7.197**/22 brd 10.0.7.255 scope global dynamic eth0

$ docker ps
CONTAINER ID    IMAGE         COMMAND      CREATED        STATUS        PORTS         NAMES
b44d7d5d3903  ubuntu:14.04    tail -f    2 seconds ago  Up 2 seconds                jovial_blackwell

$ docker exec -it b44d7d5d3903 ip addr
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
link/ether 06:58:2b:07:d5:f3 brd ff:ff:ff:ff:ff:ff
inet **10.0.7.197**/22 brd 10.0.7.255 scope global dynamic eth0
```

## ② container模式
这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。  
同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

`container模式是Kubernetes使用的网络模式`

```
$ docker run -d -P --net=bridge nginx:1.9.1
$ docker ps
CONTAINER ID  IMAGE        COMMAND   CREATED         STATUS            PORTS               NAMES
eb19088be8a0  nginx:1.9.1  nginx -g  3 minutes ago   Up 3 minutes    0.0.0.0:32769->80/tcp, 0.0.0.0:32768->443/tcp    admiring_engelbart

$ docker exec -it admiring_engelbart ip addr
...
link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
inet **172.17.0.3**/16 scope global eth0

$ docker run -it --net=container:admiring_engelbart ubuntu:14.04 ip addr
...
link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
inet **172.17.0.3**/16 scope global eth0
```

## ③ none模式
这个模式和前两个不同。在这种模式下，Docker 容器拥有自己的 Network Namespace，但是，并不为 Docker 容器进行任何网络配置。  
也就是说，这个 Docker 容器没有网卡、IP、路由等信息。需要我们自己为 Docker 容器添加网卡、配置 IP 等。

`实际上，该模式关闭了容器的网络功能，在以下两种情况下是有用的：容器并不需要网络（例如只需要写磁盘卷的批处理任务）; 你希望自定义网络`

```
$ docker run -d -P --net=none nginx:1.9.1
$ docker ps
CONTAINER ID  IMAGE          COMMAND     CREATED       STATUS         PORTS     NAMES
d8c26d68037c  nginx:1.9.1    nginx -g  2 minutes ago Up 2 minutes             grave_perlman

$ docker inspect d8c26d68037c | grep IPAddress
"IPAddress": "",
"SecondaryIPAddresses": null,
```

## ④ bridge模式
bridge 模式是 Docker 默认的网络设置，此模式会为每一个容器分配 Network Namespace、设置 IP 等，并将一个主机上的 Docker 容器连接到一个虚拟网桥上。

当 Docker server 启动时，会在主机上创建一个名为 `docker0` 的虚拟网桥，此主机上启动的 Docker 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

接下来就要为容器分配 IP 了，Docker 会从 RFC1918 所定义的私有 IP 网段中，选择一个和宿主机不同的IP地址和子网分配给 docker0，连接到 docker0 的容器就从这个子网中选择一个未占用的 IP 使用。一般 Docker 会使用 `172.17.0.0/16` 这个网段，并将`172.17.42.1/16`分配给docker0网桥。

![](http://dockone.io/uploads/article/20160503/b04b20bc12982a536ca9a35f6d5cca23.png)
![](https://upload-images.jianshu.io/upload_images/2377241-509ee9f242721eb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/444/format/webp)

```
$ docker run -d -P --net=bridge nginx:1.9.1
$ docker ps
CONTAINER ID   IMAGE                  COMMAND       CREATED         STATUS         PORTS                  NAMES
17d447b7425d   nginx:1.9.1            nginx -g   19 seconds ago  Up 18 seconds  0.0.0.0:49153->443/tcp, 0.0.0.0:49154->80/tcp  trusting_feynman
```

