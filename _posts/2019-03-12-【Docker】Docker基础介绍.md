---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. Docker概述
## ① Docker
Docker是基于Go语言实现的开源容器项目，2013年由dotCloud公司发布，Docker是近年来非常火的技术，一直备受瞩目。Docker属于Linux容器的一种封装，提供简单易用的容器接口，是目前最流行的Linux容器解决方案。Docker有3个核心概念 Image、Container、Repository。

1. Image (镜像)
    * 操作系统分为内核和用户空间，对于Linux来说内核启动后会挂载root文件系统为用户提供用户空间支持，Docker镜像就相当于一个root文件系统。
    * Docker镜像是一个特殊的文件系统，提供容器运行时所需的程序、库、资源和配置等。
    * 镜像构建的时候是一层一层构建的，前一层是后一层的基础，每一层构建完成后就不会再发生改变，后一层任何改变只发生在自己这一层。
    * 在构建镜像的时候需要特别小心，每一层应该尽量只包含需要添加的东西，任何额外的东西应该在该层构建结束的时候清理掉。
    * 分层存储的特征使得镜像很容易复用，我们可以使用构建好的镜像做为基础层定制自己需要的镜像。
2. Container (容器)
    * 镜像是静态定义，容器是镜像运行时的实体，容器可以被创建、启动、停止、删除。
    * 容器的本质是进程，与直接运行在宿主机上的进程不同的是，容器进程有自己的独立命名空间。
3. Repository (仓库)
    * 镜像仓库用于存储镜像，类似代码仓库。
    * 一个Docker Registry(注册服务器)可以包含多个Repository(仓库)，每个仓库包含多个Tag(标签)，每个标签对应一个镜像。
    * 目前Docker官方维护了一个公共镜像仓库https://hub.docker.com 超过15000个镜像，大部分镜像都可以从Docker Hub中直接找到。

## ② Docker优势
1. `更快速的交付和部署`: 使用Docker，开发人员可以使用镜像来快速构建一套标准的开发环境。开发完成后，测试和运维人员可以直接使用镜像来部署代码，可以保证测试环境和生产环境无缝衔接。Docker可以快速创建和删除容器，实现快速迭代，大量节约开发、测试、部署的时间。
2. `更高效的资源利用`: Docker容器的运行不需要额外的虚拟化管理程序支持，它是内核级的虚拟化，可以实现更高的性能，同时对资源的额外需求很低。跟传统虚拟机方式相比，要提高一到两个数量级。
3. `更轻松的迁移和扩展`: Docker容器几乎可以在任意的平台上运行，包括物理机、虚拟机、公有云、私有云、个人电脑、服务器等，同时支持主流的操作系统发行版本，这种兼容性让用户可以在不同平台之间轻松地迁移应用。
4. `更简单的更新管理`: 使用Dockerfile只需要小小的配置修改，就可以替代以往大量的更新工作。并且所有修改都以增量的方式被分发和更新，从而实现自动化并且高效的容器管理。

## ③ Docker和虚拟机对比
TODO: https://stackoverflow.com/questions/16047306/how-is-docker-different-from-a-virtual-machine

我们先来看一张图，左边是传统虚拟机实现，右边是Docker容器实现。
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_vs_vm.png?raw=true)

1. `虚拟机原理`: Hypevisor是虚拟机最重要的组成部分，它通过虚拟出一套操作系统需要的各种硬件，例如CPU、内存、I/O设备等，然后在这上面安装了一个新的操作系统，用户应用程序跑在这个新的操作系统上，应用程序只能看到Guest OS的文件和目录，所以能够起到进程隔离的作用。   
2. `Docker容器原理`: Docker取代了Hypevisor，使用Docker启动应用程序还是在宿主机上，只不过是创建进程的时候，Docker为它们加了一堆Namespace的参数。这时候，Docker里的应用进程就会觉得自己是PID Namespace里的1号进程，只能看到Mount Namespace里挂载的文件和目录，只能访问到Network Namespace里的网络设备，容器是不会创建任何实体"容器"，真正对隔离环境负责的是宿主机的操作系统。
   
虚拟机和Docker容器区别，总结起来有以下几点  
1. 虚拟机和Docker容器的最大区别是`硬件隔离`，虚拟机会虚拟出一套硬件设备，在新硬件设备上跑新的操作系统。而运行在Docker容器里面的进程则是由宿主机统一管理，只不过这些被隔离的进程拥有不同的`Namespace`参数。
2. 虚拟机需要额外的资源消耗和占用以及性能损耗，Docker容器进程由于还是运行在宿主机操作系统上，所以资源消耗和性能损耗几乎可以忽略不计。
3. 虚拟机是完全隔离的，不同虚拟机有自己的操作系统内核。Docker容器属于运行在宿主机上的一种特殊进程，所以多个容器之间共享宿主机的操作系统内核。

Docker容器特点
1. Docker容器是一种特殊的进程，核心功能是 `通过约束和修改进程的动态表现，从而为其创造出一个"边界"`，`Cgroups`是用来制造约束的主要手段，`Namespace`技术则是用来修改进程试图的主要方法。
2. Docker容器是一个`单进程`模型，用户的应用进程就是Docker容器里PID为1的进程，是后续创建的其它进程的父进程。
3. 在Windows和Mac osx上运行Docker容器，实际上是运行了一个Linux虚拟机，然后在虚拟机上运行Docker容器，所以这些Docker容器还是跑在Linux内核上。

Docker在运行应用上与传统的虚拟机方式相比具有显著优势
1. Docker容器很快，启动和停止可以在秒级实现，而传统的虚拟机方式需要数分钟。
2. Docker容器对系统资源需求很少，一台主机上可以同时运行数千个Docker容器。
3. Docker通过类似Git设计理念的操作来方便用户获取、分发和更新应用镜像，存储复用，增量更新。
4. Docker通过Dockerfile支持灵活的自动化创建和部署机制，提高工作效率，使流程标准化。

# 二. Docker Engine
Docker里面最重要的是Engine，我们看一下Docker Engine不同组件的功能。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_engine.png?raw=true)

1. `Docker Daemon`: Docker服务端守护进程用于管理所有容器，同时接收并处理Docker Client API请求。
2. `Docker Engine REST API`: Docker客户端用来和Docker Daemon通信的REST API。
3. `Docker CLI`: 与Docker Daemon通信的命令行接口，一般使用`docker`做为Docker客户端，用于管理容器、镜像、网络、卷等。

# 三. Docker 架构
Docker使用C/S架构模型，包括以下几个组件`Docker Client`、`Docker Host`、`Network`、`Storage`、`Docker Registry`

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_architecture.png?raw=true)

## ① Docker Client
Docker Client允许用户和Docker进行交互，Docker Client允许和Docker Daemon部署在同一台宿主机上，也可以连接远程的Docker Daemon，同时允许连接多个Docker Daemon。Docker Client提供了命令行接口(CLI)允许用户向Docker Daemon发起管理请求。

Docker Client最常用的命令是 构建镜像、拉取镜像、运行容器 这几个。

1. docker build ...    //构建镜像
2. docker pull ...     //拉取镜像
3. docker run ...      //运行容器

## ② Docker Host
Docker Host提供了完整的运行环境用于运行应用程序，它包括`Docker daemon`、`Images`、`Containers`、`Networks`和`Storage`。Docker daemon负责所有容器相关的操作，例如镜像拉取、构建，容器启停等。同时接收处理CLI和REST API请求，也可以和其它Docker daemon进行通信管理其服务。

## ③ Docker Objects
Docker object主要包括`Images`和`Containers`

1. `Images`: 镜像是一个用来构建容器的只读模版，通常一个镜像会依赖其他的镜像。我们可以创建自己的镜像，也可以使用仓库中已经创建好的镜像，创建镜像需要创建一个`Dockerfile`文件。
2. `Containers`: 容器是Docker架构中最终的体现形式，通过Docker daemon管理，libcontainer执行最终创建Docker容器。

## ④ Docker Network
Docker以应用程序驱动的方式实现网络并提供各种选项，同时为开发人员维护足够的抽象。有2种类型的网络可用，默认网络和用户自定义网络。默认有3种网络模式`none`、`bridge`、`host`。`none`和`host`网络模式是docker网络栈的一部分，`bridge`网络模式会自动生成网关和IP子网，属于这个网络的容器都可以通过IP进行通信。

## ⑤ Docker Storage
我们可以在容器的读写层存储数据但是它需要一个存储驱动程序，但是数据并不是持久化存储，只要容器销毁数据就会丢失，docker提供了以下4个持久化存储的选项。

1. Data Volumes: 数据卷提供了持久化存储的能力，数据卷在容器宿主机上文件系统之上非常高效。
2. Data Volume Container: 容器数据卷专用于容器托管一个卷，并将该卷挂载到其它容器，允许多个容器共享容器数据卷。
3. Directory Mounts: 直接把宿主机目录挂载到容器中，宿主机上任何目录都可以做为卷挂载。
4. Storage Plugins: 存储插件支持把数据存储到外部存储平台上

## ⑥ Docker Registry
Docker Registry是用于存储和下载镜像的服务，允许包括一个或多个镜像仓库。公共的仓库服务有Docker Hub和Docker Cloud，同时也可以搭建私有的仓库服务，常用的命令如下

1. docker push //推送镜像到仓库
2. docker pull //从仓库拉取镜像
3. docker run  //运行容器

# 四. Docker 版本
Docker CE 在 17.03 版本之前叫 [Docker Engine](https://docs.docker.com/release-notes/docker-engine/), 可以看到 Docker Engine 的版本号范围 `0.1.0 ~ 1.13.1`。

2017年3月2日, docker团队宣布企业版 Docker Enterprise Edition (EE) 发布，为了一致免费的 Docker Engine 改名为 Docker Community Edition (CE), 并且采用基于时间的版本号方案，就在这一天, Docker EE 和 Docker CE 的 `17.03` 版本发布, 这也是第一个采用新的版本号方案的版本。

Docker目前分为 `Docker CE (社区版)` 和 `Docker EE (企业版)` 这2个版本，现在Docker基于时间的发布版本，版本号格式为 `YY.MM.<patch>`，YY.MM 代表年月，patch 代表补丁号，从 0 开始。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_version.png?raw=true)

# 五. Docker OCI
Open Container Initiative (OCI)，由Linux基金会于2015年6月成立组织，旨在围绕容器格式和运行时制定一个开放的工业化标准，该组织一成立便得到了包括谷歌、微软、亚马逊、华为等一系列云计算厂商的支持。

从Docker 1.11开始，Docker容器运行已经不是简单的通过Docker daemon来启动，而是集成了containerd、runC等多个组件。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_oci.png?raw=true)

整个进程模型: dockerd -> docker-containerd -> 多个docker-containerd-shim（调用runC）-> 多个容器进程

1. Docker Engine: 最上层的容器引擎，主要是docker daemon(dockerd)守护进程，负责管理所有容器，同时接收并处理Docker Client API请求，dockerd启动的时候会启动containerd子进程。
2. docker-containerd: containerd是高性能容器运行时，容器运行时相关的程序从docker daemon剥离出来后形成了containerd。containerd向dockerd提供创建、运行容器的API，二者通过grpc进行交互，containerd管理所有容器。
3. docker-containerd-shim: containerd-shim介于containerd和runC之间的桥梁，containerd每启动一个容器就会启动一个 containerd-shim 进程，containerd 通过 containerd-shim 操作 runC，一个containerd-shim 只管理一个运行容器。
4. docker-runC: runC是一个可以用于创建和运行容器的CLI(command-line interface)工具，runc直接与容器所依赖的cgroups/linux kernel等进行交互，负责为容器配置cgroups/namespace等启动容器所需的环境，创建启动容器的相关进程。`当容器进程启动完成后runC进程会退出，容器进程交给docker-containerd-shim接管。`

为什么在容器启动过程中需要运行一个docker-containerd-shim进程呢，主要是基于以下几点考虑
1. 允许docker-runC在启动容器之后退出，也就是不必为每个容器一直运行一个容器运行时(runC)
2. 当docker-containerd 和 dockerd 都挂掉的情况下，容器依然还能运行不会中断
3. 可以在不中断容器运行的情况下升级或重启dockerd，对于生产环境有重大意义
4. 可以及时向docker-containerd发送容器的状态

我们运行一个容器，验证一下容器进程模型
1. 我们本地起一个容器  
   `$ docker run -it --memory 100M --cpu-period=100000 --cpu-quota=50000 --cpuset-cpus 1 --device /dev/sda:/dev/sda --device-read-bps /dev/sda:1mB --device-write-bps /dev/sda:1mB ubuntu`

2. 查看容器在物理机PID，找到PID为`36992`  
   `$ docker inspect 5508e831f91581c15742bef716dc1a602450f2b4f0d994e4ecabb58189fec02f | grep Pid`
   
3. 查看进程树，发现符合上述的进程模型
   ```
   1). 查看pid 36992，容器进程命令是 /bin/bash，父进程为 36975
       [root@localhost root]# ps -ef | grep 36992
       root   36992 36975  0  5月11 pts/3   00:00:00 /bin/bash
   2). 查看pid 36975，实际是一个 docker-containerd-shim 进程，父进程为 8175
       [root@localhost root]# ps -ef | grep 36992
       root   36975  8175  0 5月11 ?      00:00:00 docker-containerd-shim 13a918bb87f0482d5fddd8c9ad43521ae1d9e499d93b032d5e15ff1ccad3153f /var/run/docker/libcontainerd/13a918bb87f0482d5fddd8c9ad43521ae1d9e499d93b032d5e15ff1ccad3153f docker-runc
   3). 查看pid 8175，实际是一个 docker-containerd 进程，父进程为 8151
       root   8175  8151  0 2月21 ?       18:08:37 docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --shim docker-containerd-shim --runtime docker-runc
   4). 查看pid 8151，实际是一个 dockerd 进程，父进程为 1 (init进程)
       root      8151     1  5 2月21 ?       4-16:03:24 dockerd --insecure-registry=harbor.cloud.m.com --graph=/var/lib/docker --log-opt max-size=100m --log-opt max-file=3 --iptables=false --storage-driver devicemapper --storage-opt dm.thinpooldev=/dev/mapper/docker-thinpool --storage-opt dm.fs=xfs --storage-opt dm.use_deferred_deletion=true --storage-opt dm.use_deferred_removal=true --storage-opt dm.use_deferred_deletion=true --dns 114.114.114.114 --dns-search default.svc.cluster.local --dns-search svc.cluster.local --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2
   ```

# 六. Docker 好图
1. docker CLI全图
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-cli.png?raw=true)
2. docker 状态图
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-status.png?raw=true)

