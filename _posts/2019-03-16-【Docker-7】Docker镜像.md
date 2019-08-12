---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

TODO (@玄苦)

# 一. 镜像介绍
Docker镜像是通过 Dockerfile 构建出来的，Docker镜像在设计上引入了`层`的概念，也就是说用户制作镜像的每个步骤都会生成一个层。当镜像构建完成后，因为`storage-driver`的不同镜像存储的目录也不一样，通常在`dockerd`启动时候会设定 `--storage-driver`。

最常见的storage-driver有以下3种
1. `aufs`: ubuntu使用aufs做为storage-driver
    * /var/lib/docker/aufs/diff/<id>: 存储镜像内容
    * /var/lib/docker/repositories-aufs: JSON文件包含本地镜像信息
2. `devicemapper`: Redhats使用centos做为storage-driver，devicemapper把镜像和容器存储在自己的虚拟设备
    * /var/lib/docker/devicemapper/metadata: 镜像meta信息
    * /var/lib/docker/devicemapper/mnt: 每个容器RootFs真实挂载目录
3. `overlay2`

我们可以知道Docker镜像本质是 `挂载在容器根目录上，用来作为容器进程提供隔离后执行环境的文件系统，称为RootFS`，镜像是静态定义，容器是镜像运行时表现。

Docker镜像是通过Dockerfile构建出来的，Docker镜像在设计上引入了`层`的概念，也就是说用户制作镜像的每个步骤都会生成一个层，Docker使用UnionFS（Union File System）将镜像多个层联合挂载到同一个目录下，这个目录会被挂载到容器根目录上，做为容器进程的根文件系统使用，即RootFS也就是镜像。

RootFS由三个部分组成 `镜像层`、`init层`、`容器层`
1. 镜像层: 
2. init层:
3. 容器层: 

ubuntu系统下挂载目录为 `/var/lib/docker/aufs/mnt/{mount_id}`，centos系统下挂载目录为 `/var/lib/docker/devicemapper/mnt/{mount_id})`，最后使用pivot_root或chroot切换容器根目录到RootFS挂载点，成为一个完整操作系统供容器使用。


# 二. 常用命令
参考

1. https://mp.weixin.qq.com/s/MOtGXCRwTWKfH7KuzvZ9ww
2. https://www.sandtable.com/reduce-docker-image-sizes-using-alpine/

# 三. 镜像构建

# 四. Dockerfile最佳实践


