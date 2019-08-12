---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. 概述
TODO (@玄苦)

参考  
1. https://mp.weixin.qq.com/s/ysAvzhrZ5vh_i2ek8nH_oQ  
2. https://deepzz.com/post/the-docker-volumes-basic.html  
3. https://docs.docker.com/v17.09/engine/admin/volumes/  

主要包括以下3类  
1. Volumes
2. Bind Mounts
3. tmpfs Mounts

生产环境中使用Docker的过程中，往往需要对数据进行持久化，或者需要在多个容器之间进行数据共享。容器中管理数据主要有两种方式:
1. 数据卷(Data Volumes):容器内数据直接映射到本地主机环境;
2. 数据卷容器(Data Volume Containers):使用特定容器映射到本地主机环境;

# 二. Data Volumes
Data Volumes（数据卷）是一个可供容器使用的特殊目录，它将主机操作系统目录直接映射进容器，类似与Linux中的monut操作。

数据卷可以提供很多有用的特性  
1. 数据卷可以在容器之间共享和重用，容器间传递数据将变得高效方便
2. 对数据卷内数据的修改会立马生效，无论是容器内操作还是本地操作
3. 对数据卷的更新不会影响镜像，解藕了应用和数据
4. 卷会一直存在，直到没有容器使用，可以安全地卸载它

Data Volumes（数据卷）创建: 在用`docker run`命令的时候，使用`-v`标记可以在容器内创建一个数据卷，多次使用`-v`标记可以创建多个数据卷。

Volume机制允许将宿主机上指定的目录或文件挂载到容器里面进行读取和修改操作，有2种Volume声明方式，可以把宿主机目录挂载进容器中，两种方式本质是相同的都是把一个宿主机的目录挂载到容器的/test目录。
```
$ docker run -v /test ...
$ docker run -v /home:/test ...
```
第一种方式由于没有显式的声明宿主机目录，那么会默认在宿主机上创建一个临时目录/var/lib/docker/volumes/[VOLUME_ID]/_data然后挂载到/test目录。

容器在创建之后，仅管已经开启了Mount Namespace 如果没有执行pivot_root或chroot，那么容器内能看到整个宿主机文件系统。容器镜像各个层会保存在宿主机/var/lib/docker/aufs/diff目录下，在进程启动之后会被联合挂载到宿主机/var/lib/docker/aufs/mnt目录中。所以我们只需要在容器创建之后，pivot_root或chroot执行之前，把宿主机指定的目录挂载到宿主机/var/lib/docker/aufs/mnt/[读写层ID]/[目录名] 下，那这个Volume就算完成。

在执行这个挂载的时候，由于容器已经创建，也就是Mount Namespace已经开启了。所以这个挂载只在这个容器内可见，在宿主机上是看不到这个挂载点的，因此就保证了容器的隔离性不会被Volume打破。

Volume挂载用到的技术是Linux的`绑定挂载(bind mount)`技术，允许你将一个目录或文件挂载到一个指定的目录下，这时候对挂载点进行的操作只是发生在被挂载
的目录上，原挂载点的内容不受影响。例如把/home挂载到/test目录下，相当于把/test的inode重定向到/home目录下，对/test目录的任何操作实际修改的是/home目录。
![](https://static001.geekbang.org/resource/image/95/c6/95c957b3c2813bb70eb784b8d1daedc6.png)

容器Volume挂载点内的信息并不会被docker commit提交掉，因为docker commit是发生在宿主机空间上，但是由于Mount Namespace Volume挂载只在容器内可见，宿主机并不知道挂载点存在，因此对于宿主机来说这个挂载点目录始终为空。

# 三. Data Volume Containers
如果需要在多个容器之间共享一些持续更新的数据，最简单的方式是使用Data Volume Containers（数据卷容器），数据卷容器也是一个容器，但是他的目的是专门用来提供数据卷供其他容器挂载。

TODO（@玄苦）

