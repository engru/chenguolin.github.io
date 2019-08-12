---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. Rootfs 概述
为了能够让容器的根目录看起来更真实，我们一般会在容器根目录挂载一个完整的操作系统的文件系统，例如Ubuntu 16.04的ISO。这样，当我们在容器启动之后在容器内执行`ls /`命令就是查看根目录内容，实际上就是查看Ubuntu 16.04文件系统所有目录和文件。

`这个挂载在容器根目录用来为容器进程提供隔离环境的文件系统，就是Rootfs（根文件系统），俗称镜像`。
一个常见的Rootfs，通常会包括以下一些文件和目录，这和常规的Linux文件系统并无太大差别。
```
# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
```
所以当我们进入到容器内执行`ls`、`sh`等shell命令的时候实际上调用的是Rootfs`/bin`目录下的可执行文件，这和宿主机上的`/bin`目录是完全不同的。

`Rootfs`只包含操作系统相关的文件、配置和目录，并不包含操作系统的内核。所以，同一个宿主机上的所有容器都是共享宿主机操作系统内核的。因此，如果容器如果需要修改内核相关的参数，就可能会影响所有的容器。

`Rootfs`最重要的一个特点是: 给容器提供一致性，由于Rootfs不只打包应用程序，还包含操作系统的文件、配置和目录，这说明Rootfs包含应用以及应用的所有依赖（除了内核）。因此无论是在本地还是在远端，只需要提供一个容器镜像都能保证应用运行在一致的环境中。

通过结合使用Mount Namespace和Rootfs，容器就能够为进程提供隔离的操作系统运行环境，Linux默认会使用pivot_root和chroot来切换根目录`/`到Rootfs的挂载点。

# 二. UnionFS
Docker在设计镜像的时候引入了层`layer`的概念，用户制作镜像的每一步都会生成一个层，也就是一个增量的Rootfs，最终所有层如何变成一个完整的Rootfs则是使用`UnionFS(Union File System)`联合文件系统。

UnionFS最主要的功能是将多个不同位置的目录联合挂载到同一个目录下，例如有2个子目录的d1和d2，可以使用UnionFS将这两个目录挂载到d3目录下。
常见的UnionFS实现有`aufs`, `device mapper`, `btrfs`, `overlayfs`, `vfs`, `zfs`。aufs是ubuntu常用的，device mapper 是 centos，overlayfs ubuntu 和 centos 都会使用，现在最新的 docker 版本中默认两个系统都是使用的 overlayfs。

下面我们使用aufs举例，UnionFS如何实现联合挂载。可以看到使用aufs联合挂载之后，d3目录里面相同文件名的文件只有一份，以此同时如果修改了d3目录下的文件，则d1、d2目录里对应文件也会变更。
```
$ tree
.
├── d1
│   ├── 1
│   └── common
└── d2
    ├── 2
    └── common
    
$ mount -t aufs -o dirs=./d1:./d2 none d3
$ tree
.
├── d3
│   ├── 1
│   ├── 2
│   └── common
```

# 三. 镜像
当我们启动一个ubuntu容器的时候默认会从docker hub上拉取ubuntu镜像到本地，这个ubuntu镜像实际上就是一个包含操作系统文件、配置和目录的Rootfs。

1. 例如我们启动一个ubuntu容器  
   `$ docker run -it  -d ubuntu`
2. 查看ubuntu镜像Rootfs构成
   ```
   "RootFS": {
       "Type": "layers",
       "Layers": [
            "sha256:762d8e1a60542b83df67c13ec0d75517e5104dee84d8aa7fe5401113f89854d9",
            "sha256:e45cfbc98a505924878945fdb23138b8be5d2fbe8836c6a5ab1ac31afd28aa69",
            "sha256:d60e01b37e74f12aa90456c74e161f3a3e7c690b056c2974407c9e1f4c51d25b",
            "sha256:b57c79f4a9f3f7e87b38c17ab61a55428d3391e417acaa5f2f761c0e7e3af409"
       ]
   }
   ```
   可以看到这个ubuntu镜像实际由4个层组成，每一层都是增量Rootfs
3. 当我们启动容器的时候会把这4层联合挂载到一个统一的挂载点上，由于使用UnionFS技术不同，则挂载的目录会不同  
   ```
   aufs: /var/lib/docker/aufs/mnt/{mount_id}
   devicemapper: /var/lib/docker/devicemapper/mnt/{mount_id}
   
   最新的docker版本推荐使用overlay2做为storage driver
   ```
4. 需要注意的是在多层联合挂载到统一挂载点上的时候，上层文件会覆盖下层文件。  
   举个例子，例如d60e01b37e74f12aa90456c74e161f3a3e7c690b056c2974407c9e1f4c51d25b 这一层有个文件`/tmp/1.txt`，内容为`例如d60e01b37e74f12aa90456c74e161f3a3e7c690b056c2974407c9e1f4c51d25b 1.txt`。  
   e45cfbc98a505924878945fdb23138b8be5d2fbe8836c6a5ab1ac31afd28aa69 这一层也有一个文件`/tmp/1.txt`，内容为`e45cfbc98a505924878945fdb23138b8be5d2fbe8836c6a5ab1ac31afd28aa69 1.txt`。  
   由于两层之间有同名文件，但是内容不同，UnionFS做法是上层文件覆盖下层文件，最终我们看到`/tmp/1.txt`文件的内容为`e45cfbc98a505924878945fdb23138b8be5d2fbe8836c6a5ab1ac31afd28aa69 1.txt`。

由于容器镜像分层设计的原因，所有基础层可以做为共享层存在，容器镜像所需要的空间也就比每个镜像总和要小，同时每次拉取、推送镜像也会快很多。

# 四. COW
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker_rootfs_layer.png?raw=true)

如上图所示，Rootfs实际上由3个部分组成

1. 第一部分 `只读层(镜像层)`  
   `只读层`也称为镜像层是Rootfs最下面的层，一般称为基础镜像。
2. 第二部分 `init层`  
   `init层`位于只读层和读写层中间，init层是Docker容器单独生成的的一个内部层，用于存放/etc/hosts、/etc/resolv.conf等信息，由Docker引擎在容器启动之前做好。
3. 第三部分 `读写层(容器层)`  
   `读写层`也称为容器层，位于Rootfs最上层，容器启动之后会在最上层加上读写层，读写层是专门用来存放修改Rootfs之后的增量，无论是增、删、改都保存在这一层
   
`读写层保存的是容器镜像变化的部分，并不会对只读层做任何修改。在容器中读取、修改、删除文件的时候，会从上到下查找此文件，找到文件后就复制到读写层，然后做修改，修改之后就会覆盖掉下层的文件，这种方式也被称为COW (Copy-On-Write)。`

# 五. Docker 使用Rootfs
centos操作系统下，我们可以看一下使用docker运行一个容器后，Rootfs的挂载情况

1. 启动一个容器，容器id为 23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f  
   `$ docker run -it  -d ubuntu`
2. 查看Rootfs的挂载id，挂载点id为 e99e450494c3d9f902f60a15a2216ccb1475f8df6144f402434a946fc2352452  
   `$ cat /var/lib/docker/image/devicemapper/layerdb/mounts/23649656f1f33024cc02f414e78a9a43902fcd40ed829744664fa1fb159ed82f/mount-id`
3. 查看挂载点
   ```
   $ ls -lsrt /var/lib/docker/devicemapper/mnt/e99e450494c3d9f902f60a15a2216ccb1475f8df6144f402434a946fc2352452
   4 -rw-------.  1 root root  64 4月   1 17:45 id
   0 drwxr-xr-x. 21 root root 242 5月  12 23:09 rootfs
   ```
   
