---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. 概述
日常开发过程中 `日志打印` 是必不可少的环节，但程序日志打印却面临2个问题，`日志打印到哪里？` 和 `日志如何备份？`。

1. 日志打印到哪里？
    + 在Docker、K8s等容器化平台还未兴起的时候，我们的应用都是直接部署在 `物理机` 上，大部分程序日志打印都是输出到 `文件`。
    + 在全面容器化时代到来之后，我们的应用变成部署在K8s集群底层使用Docker运行，这个时候大部分程序日志可以直接打印到 `stdout或stderr`，由Docker log-driver接管，但是少部分应用可能还需要把日志打印到容器内文件。
2. 日志如何备份？
    + 日志打印到 stdout、stderr 的时候，我们不需要考虑备份日志问题
    + 日志打印到 文件 的时候，我们就需要考虑日志如何备份，备份就会面临日志文件滚动切割问题。这些问题可以由业务方通过代码来控制，Java应用可以使用log4j，Golang应用可以使用Zap。除此之外，也可以由运维工具来实现，例如 `logratate`。

# 二. inode
我们知道在 Linux 下一切设备皆 `文件`，而 `inode` 正是用来描述这个文件的。

`inode` 指的是用来描述文件、目录的数据结构，每个文件有一个 `inode`，每个 `inode` 由一个整数编号标识。`inode` 存储当前文件的所有者、访问权限、以及文件类型等信息。通过 `inode` 编号，内核文件系统驱动可以访问到文件内容。

Linux下，我们可以使用 `ls -i` 或 `stat` 命令查看某个文件的inode

```
[cgl@test.com ~]$ ls
4 drwxrwxr-x 2 cgl cgl     4096 Jun 22 21:38 tmp
4 -rw-rw-r-- 1 cgl cgl     1067 Jul 15 11:33 log

[cgl@test.com ~]$ ls -i log
1315851 log

[cgl@test.com ~]$ ls -i tmp
1315051 tmp.yaml

[cgl@test.com ~]$ stat log
  File: `log'
  Size: 1067      	Blocks: 8          IO Block: 4096   regular file
Device: 803h/2051d	Inode: 1315851     Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  890/     cgl)   Gid: (  891/     cgl)
Access: 2019-07-26 09:57:39.764554472 +0800
Modify: 2019-07-15 11:33:01.067210769 +0800
Change: 2019-07-15 11:33:01.068210815 +0800

[cgl@test.com ~]$ stat tmp
  File: `tmp'
  Size: 4096      	Blocks: 8          IO Block: 4096   directory
Device: 803h/2051d	Inode: 1316019     Links: 2
Access: (0775/drwxrwxr-x)  Uid: (  890/     cgl)   Gid: (  891/     cgl)
Access: 2019-07-26 10:02:24.024454275 +0800
Modify: 2019-06-22 21:38:06.204658098 +0800
Change: 2019-06-22 21:38:06.204658098 +0800
[cgl@BJSH-VM-132-29.meitu-inc.com ~]$
```

由上可知，类Unix操作系统内部不直接使用文件名，而使用 `inode` 号码来标识文件，文件名只是一个别名。所以，当我们打开一个文件的时候操作系统会先找到这个文件名对应的 `inode` 编号，然后通过 `inode` 编号获取 `inode` 信息，最后根据 `inode` 信息找到文件数据所在的 `block` 读出数据。

# 三. logrotate

