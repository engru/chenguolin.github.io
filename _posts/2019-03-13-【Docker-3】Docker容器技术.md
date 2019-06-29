---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. 核心技术
目前Docker底层依赖的核心技术主要包括Linux操作系统的Namespace(命名空间)、Cgroups(控制组)、Rootfs(根文件系统)。  
一个`容器`实际上是一个由 Linux Namespace、Linux Cgroups和Rootfs三种技术构建出来的进程隔离环境。

## ① Namespace
`Namespace`是Linux提供的一种内核级别环境隔离的方法，是Linux创建新进程的一个可选参数，Linux操作系统提供了`PID`、`Mount`、`UTS`、`IPC`、`Network`、`User`、`Cgroup`这几个Namespace。Linux每个进程的Namespace在/proc/[pid]/ns目录下都有对应的虚拟文件。

1. `PID Namespace`: 用于让被隔离的进程只看到当前Namespace里的进程信息，例如我们启动一个容器`docker run -it busybox /bin/sh` 使用`ps`命令会发现/bin/sh进程PID为1。
2. `Mount Namespace`: 用于让被隔离的进程只看到当前Namespace里的挂载点信息。
3. `Network Namespace`: 用于让被隔离的进程只看到当前Namespace里的网络设备信息。

`Namespace`是Docker容器中用来实现"隔离"的技术手段，Namespace技术实际上是修改了应用进程看待整个计算机的"试图"，也就是它被操作系统限制了只能看到某些指定的内容，但对于宿主机来说这些被隔离的进程与其它进程是没有区别的。

`Docker run在创建容器进程的时候，实际上是指定了这个进程所要启用的一组 Namespace 参数，这样容器启动之后就只能看到当前 Namespace 所限定的资源、文件、设备、状态等等，对于宿主机或者其他不相关的程序它就看不到了。`

`docker exec`命令用于进入某一个容器，实现原理是通过把进程加入到容器进程已有的Namespace当中，从而达到进入某个容器的目的。例如`docker exec -it 4ddf4638572d /bin/bash`通过把`/bin/bash`进程加入到`4ddf4638572d`容器进程的Namespace中，这样我们在shell里面看到的就是`4ddf4638572d`容器进程的视图。

## ② Cgroups
我们知道Docker容器进程是运行在宿主机上的特殊进程，与其它的进程是平等竞争关系。那这会带来一个问题就是这些进程有可能一直获取不到资源(CPU、内存等)，也有可能把所有的资源都独占，这个问题可以通过Linux Cgroups来解决。

Linux `Cgroups`是Google工程师在2006年发起的，全称为`Linux Control Groups`，是Linux内核用来限制一个进程组能够使用的资源`上限`，资源包括CPU、内存、磁盘、网络带宽等。

在Linux中Cgroups给用户暴露出来的接口是`文件系统`，它是以文件和目录的形式组织在操作系统的`/sys/fs/cgroup`路径下的，使用`mount -t cgroup`命令可以查看Cgroups的挂载目录，可以看到`/sys/fs/cgroup`目录下有很多子目录例如`cpu`、`cpuset`、`memory`，这些都是当前可以被Cgroups限制的资源种类。
```
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,perf_event)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,net_cls,net_prio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,freezer)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,memory)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpu,cpuacct)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuset)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,rdma)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,blkio)
```

我们可以做一个测试，了解Cgroups是如何限制进程资源使用的。
1. 先在/sys/fs/cgroup/cpu目录下创建一个`控制组`名为containers  
   `$ mkdir /sys/fs/cgroup/cpu/containers` 
2. 进入/sys/fs/cgroup/cpu/containers目录，发现会自动生成资源限制文件   
   ```
   cgroup.clone_children  
   cgroup.event_control  
   cgroup.procs  
   cpuacct.stat  
   cpuacct.usage  
   cpuacct.usage_percpu  
   cpu.cfs_period_us  
   cpu.cfs_quota_us  
   cpu.rt_period_us  
   cpu.rt_runtime_us  
   cpu.shares  
   cpu.stat  
   notify_on_release  
   tasks  
   ```  
 3. 执行shell命令`$ while : ; do : ; done &`，使用top命令，发现当前进程直接把CPU跑到100% （因为我们没有对进程做任何CPU使用限制，所以跑满了单核）
 4. Cgroups配置CPU限制  
    * 修改cpu.cfs_quota_us文件，让每个进程只能使用20%的CPU时间  
      `echo 20000 > cpu.cfs_quota_us` (为什么是20000，因为默认cpu.cfs_period_us值为100000表示100ms，我们设置20000就表示20ms占比20%)  
    * 我们把PID写入tasks文件  
      `echo 15150 >> tasks`  
5. 再次使用top命令查看，发现CPU使用降到了20%
   
Linux Cgroups实现简单的理解就是在`/sys/fs/cgroup/`不同子目录上加上一层资源限制文件，对于Docker容器来说，只需要在不同子目录上创建一个`docker`控制组，每个Docker容器进程启动的时候就在`docker`目录下创建一个名为container name的子目录，在里面配置对应的资源限制，并把容器进程PID写入tasks文件即可。

## ③ Rootfs
为了让容器进程的`/`目录看起来更真实，我们一般会在容器的根目录下挂载一个完整的操作系统文件，例如Ubuntu、Centos、Debian等。Linux的chroot与pivot_root可以用来实现切换文件系统根目录，Docker内部优先使用pivot_root，找不到再使用chroot。

`挂载在容器根目录上，用来作为容器进程提供隔离后执行环境的文件系统，专业名为Rootfs(根文件系统)。`

Rootfs有以下几个特点
1. 常见的 Rootfs 根目录通常会包含这些子目录`bin dev etc home lib lib64 mnt opt proc root run sbin sys tmp usr var`。
2. Rootfs 只会包含操作系统所需要的文件、配置和目录，并不包含操作系统内核。同一台宿主机上的容器，都是共享操作系统内核的，因此涉及到内核参数的修改会影响所有容器。
3. Rootfs 包含了用户应用运行所需要的所有依赖，提供深入到操作系统级别的一致性，保证在任何机器上用户应用最终运行的操作系统环境都是一致的。 (操作系统内核可能存在差别)

Docker镜像在设计上引入了`层`的概念，也就是说用户制作镜像的每个步骤都会生成一个层，也就是一个增量的Rootfs，Docker使用`UnionFS（Union File System）`来实现，用于将多个不同位置的目录联合挂载到同一个目录下。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_rootfs_layer.png?raw=true)

如上图所示，Rootfs实际上由3个部分组成

1. 第一部分 `只读层(镜像层)`  
   只读层也称为镜像层是 Rootfs 最下面的层，一般称为基础镜像。
2. 第二部分 `init层`  
   init层位于只读层和读写层中间，init层是Docker容器单独生成的的一个内部层，用于存放/etc/hosts、/etc/resolv.conf等信息，由Docker引擎在容器启动之前做好。
3. 第三部分 `读写层(容器层)`  
   读写层也称为容器层位于 Rootfs 最上层，容器启动之后会在最上层加上读写层，读写层是专门用来存放修改 Rootfs 之后的增量，无论是增、删、改都保存在这一层，读写层保存的是容器镜像变化的部分，并不会对镜像本身做任何修改。在容器中读取、修改、删除文件的时候，会从上到下查找此文件，找到文件后就复制到容器层，然后做修改，修改之后就会覆盖掉下层的文件。

由于 Rootfs 是分层设计，使得多个容器可以共享资源，所以每次镜像拉取、推送的内容比原先完整操作系统要小的多。(并不是层数越多越好，层数越多会导致镜像臃肿) 
从下面这个例子可以看出，由于本地已经有了ubuntu其它版本的镜像，因此在拉取14.04版本ubuntu镜像的时候，基础层是不需要再次拉取的。
```
$ docker pull ubuntu:14.04
latest: Pulling from library/ubuntu:14.04
898c46f3b1a1: Already exists
63366dfa0a50: Already exists
041d4cd74a92: Already exists
6e1bee0f8701: Already exists
...
```

`docker commit`命令可以提交一个正在运行的容器，实际上就是容器起来后把最上层的`读写层`加上原先容器镜像的`只读层`打包组成一个新的镜像并提交，这个新的镜像被其他人使用之后就变成`只读层`。

# 二. 总结
从上面的介绍，我们可以知道Docker容器最核心的原理是 `为待创建的用户进程执行以下3个操作`
1. 启用一堆Linux Namespace参数，包括PID Namespace、Network Namespace、Mount Namespace等等
2. 使用Cgroups配置进程资源限制
3. 使用pivot_root或chroot切换进程根目录，切换到Rootfs挂载点(/var/lib/docker/aufs/mnt/ 或者 /var/lib/docker/devicemapper/mnt/)

容器创建过程由容器初始化进程(dockerinit)负责完成根目录准备、挂载设备和目录、配置hostname等一序列需要在容器内进行的初始化操作，最后它通过`execv()`系统调用，让应用进程代替自己成为PID为1的进程。

例如，我们运行一个容器 `docker run -p 4000:80 helloworld python app.py`

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_rootfs_layer.png?raw=true)

从上面这个结构我们可以看出，一个正在运行的Linux容器可以分为2部分

1. 联合挂载在/var/lib/docker/aufs/mnt/ 或者 /var/lib/docker/devicemapper/mnt/ 目录下的Rootfs(容器镜像)，是容器的静态试图
2. 由Namespace和Cgroups构成的进程隔离环境，我们称为容器运行时，是容器的动态试图

`Docker的核心价值是加快软件交付的效率、提高生产力，实现了应用与运行环境的解耦，很多业务应用负载都可以进行容器化。`

# 三. Docker使用
我们使用centos操作系统Docker启动一个容器来举例说明Linux容器的实现原理

1. 启动容器，container id为 23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f  
   `$ docker run -it  -d ubuntu`
2. 查看容器相关的文件
   ```
   $ find / -name "23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f"
   /run/runc/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f    //runC目录
   /run/docker/libcontainerd/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f   //containerd-shim目录
   /run/docker/libcontainerd/containerd/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /www/docker/containers/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f   //container目录
   /www/docker/image/devicemapper/layerdb/mounts/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f  //RootFS挂载点
   /sys/fs/cgroup/blkio/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f   //cgroups
   /sys/fs/cgroup/cpuset/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/cpu,cpuacct/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/pids/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/memory/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/freezer/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/devices/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/hugetlb/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/net_cls,net_prio/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/perf_event/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   /sys/fs/cgroup/systemd/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   ```
3. 查看runc对应的目录信息
   ```
   $ ls -lsrt /run/runc/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   28 -rw-r--r--. 1 root root 24742 5月  12 23:09 state.json
   
   $ cat state.json
   {
    "cgroup_paths": {
        "blkio": "/sys/fs/cgroup/blkio/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "cpu": "/sys/fs/cgroup/cpu,cpuacct/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "cpuacct": "/sys/fs/cgroup/cpu,cpuacct/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "cpuset": "/sys/fs/cgroup/cpuset/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "devices": "/sys/fs/cgroup/devices/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "freezer": "/sys/fs/cgroup/freezer/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "hugetlb": "/sys/fs/cgroup/hugetlb/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "memory": "/sys/fs/cgroup/memory/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "name=systemd": "/sys/fs/cgroup/systemd/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "net_cls": "/sys/fs/cgroup/net_cls,net_prio/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "net_prio": "/sys/fs/cgroup/net_cls,net_prio/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "perf_event": "/sys/fs/cgroup/perf_event/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
        "pids": "/sys/fs/cgroup/pids/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f"
    },
    "config": {
        ...     //省略内容
        "labels": [
            "bundle=/run/docker/libcontainerd/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f"
        ],
        "no_new_keyring": false,
        "no_pivot_root": false,
        "parent_death_signal": 0,
        "readonlyfs": false,
        "rootPropagation": 278528,
        "rootfs": "/var/lib/docker/devicemapper/mnt/e99e450494c3d9f902f60a15a2216ccb1475f8df6144f402434a946fc2352452/rootfs",
        "version": "1.0.0-rc2-dev"
    },
    "created": "2019-05-12T15:09:19.615217106Z",
    "external_descriptors": [
        "/dev/null",
        "/dev/null",
        "/dev/null"
    ],
    "id": "23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f",
    "init_process_pid": 48289,
    "init_process_start": "693752862",
    "namespace_paths": {
        "NEWIPC": "/proc/48289/ns/ipc",
        "NEWNET": "/proc/48289/ns/net",
        "NEWNS": "/proc/48289/ns/mnt",
        "NEWPID": "/proc/48289/ns/pid",
        "NEWUSER": "/proc/48289/ns/user",
        "NEWUTS": "/proc/48289/ns/uts"
    }
   }
   ```
   
   从以上信息可以知道  
   1. 容器进程的PID为48289  
   2. Rootfs挂载点为/var/lib/docker/devicemapper/mnt/e99e450494c3d9f90...  
   3. 进程namespace、cgroups对应的目录
4. 查看容器进程Namespace信息
   ```
   $ ls -lsrt /proc/48289/ns/
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:09 net -> net:[4026533582]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 uts -> uts:[4026533578]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 user -> user:[4026531837]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 pid_for_children -> pid:[4026533580]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 pid -> pid:[4026533580]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 mnt -> mnt:[4026533577]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 ipc -> ipc:[4026533579]
   0 lrwxrwxrwx. 1 root root 0 5月  12 23:16 cgroup -> cgroup:[4026531835]
   ```
5. 查看容器进程cgroups对应配置
   ```
   $ ls -lsrt /sys/fs/cgroup/memory/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   $ ls -lsrt /sys/fs/cgroup/cpu,cpuacct/docker/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   ...
   ```
6. 查看Rootfs挂载点
   ```
   $ ls -lsrt /var/lib/docker/devicemapper/mnt/e99e450494c3d9f902f60a15a2216ccb1475f8df6144f402434a946fc2352452/rootfs
   0 drwxr-xr-x.  8 root root   96 5月  23 2017 lib
   0 drwxr-xr-x.  2 root root    6 4月  24 2018 sys
   0 drwxr-xr-x.  2 root root    6 4月  24 2018 proc
   0 drwxr-xr-x.  2 root root    6 4月  24 2018 home
   0 drwxr-xr-x.  2 root root    6 4月  24 2018 boot
   0 drwxr-xr-x. 10 root root  105 3月   8 05:00 usr
   0 drwxr-xr-x.  2 root root    6 3月   8 05:00 srv
   0 drwxr-xr-x.  2 root root    6 3月   8 05:00 opt
   0 drwxr-xr-x.  2 root root    6 3月   8 05:00 mnt
   0 drwxr-xr-x.  2 root root    6 3月   8 05:00 media
   0 drwxr-xr-x.  2 root root   34 3月   8 05:00 lib64
   0 drwxr-xr-x. 11 root root  139 3月   8 05:01 var
   0 drwx------.  2 root root   37 3月   8 05:01 root
   4 drwxr-xr-x.  2 root root 4096 3月   8 05:01 bin
   0 drwxrwxrwt.  2 root root    6 3月   8 05:01 tmp
   4 drwxr-xr-x.  2 root root 4096 3月  12 08:20 sbin
   0 drwxr-xr-x.  5 root root   58 3月  12 08:20 run
   0 drwxr-xr-x.  4 root root  182 5月  12 23:09 dev
   4 drwxr-xr-x. 29 root root 4096 5月  12 23:09 etc
   ```
7. 查看容器相关信息
   ```
   $ ls -lsrt /www/docker/containers/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f
   4 drwx------. 2 root root 4096 5月  12 23:09 checkpoints
   4 -rw-r--r--. 1 root root   71 5月  12 23:09 resolv.conf.hash
   4 -rw-r--r--. 1 root root  162 5月  12 23:09 resolv.conf
   4 -rw-r--r--. 1 root root  174 5月  12 23:09 hosts
   4 -rw-r--r--. 1 root root   13 5月  12 23:09 hostname
   0 drwxrwxrwt. 2 root root   40 5月  12 23:09 shm
   0 -rw-r-----. 1 root root    0 5月  12 23:09 23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f-json.log
   4 -rw-r--r--. 1 root root 1148 5月  12 23:09 hostconfig.json
   4 -rw-r--r--. 1 root root 2559 5月  12 23:09 config.v2.json
   ```
   
