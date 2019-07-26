---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Linux
---

# 一. 概述
日常开发过程中 `日志打印` 是必不可少的环节，但程序日志打印却面临2个问题。

1. 日志打印到哪里？
    + 在Docker、K8s等容器化平台还未兴起的时候，我们的应用都是直接部署在 `物理机` 上，大部分程序日志打印都是输出到 `文件`。
    + 在全面容器化时代到来之后，我们的应用变成部署在K8s集群底层使用Docker运行，这个时候大部分程序日志可以直接打印到 `stdout或stderr`，由Docker log-driver接管，但是少部分应用可能还需要把日志打印到容器内文件。
2. 日志如何备份？
    + 日志打印到 stdout、stderr 的时候，我们不需要考虑备份日志问题。
    + 日志打印到 文件 的时候，我们就需要考虑日志如何备份，备份就会面临日志文件滚动切割问题。这些问题可以由业务方通过代码来控制，Java应用可以使用log4j，Golang应用可以使用Zap。除此之外，也可以由运维工具来实现，例如 `logratate`。

# 二. inode
我们知道在 Linux 下一切设备皆 `文件`，而 `inode` 则是用来描述文件数据结构。每个文件有一个 `inode`，每个 `inode` 由一个整数编号标识。`inode` 存储当前文件的所有者、访问权限、以及文件类型等信息。通过 `inode` 编号，内核文件系统驱动可以访问到文件内容。

Unix/Linux操作系统内部不是直接使用文件名，而是使用 `inode` 编号来标识文件，文件名只是一个别名。  
所以，当我们打开一个文件的时候操作系统会先找到这个文件名对应的 `inode` 编号，然后通过 `inode` 编号获取 `inode` 信息，最后根据 `inode` 信息找到文件数据所在的 `block` 读出数据。

Linux下，我们可以使用 `ls -i` 或 `stat` 命令查看某个文件的inode。

```
[cgl@test.com ~]$ ls
4 drwxrwxr-x 2 cgl cgl     4096 Jun 22 21:38 tmp    //目录
4 -rw-rw-r-- 1 cgl cgl     1067 Jul 15 11:33 log    //文件

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

了解了 `inode` 一些基础知识后，我们来看几个问题

1. 我们使用shell脚本重定向输出到 `test_log` 文件，在运行过程中 `rm` 该文件
```
[cgl@test.com ~]$ sh run.sh > test_log
[cgl@test.com ~]$ ls -i test_log
1315128 test_log
[cgl@test.com ~]$ find -inum 1315128 -exec head -n 2 {} \;   //根据inode查看文件内容
hello cgl ~
hello cgl ~
[cgl@test.com ~]$ rm test_log
[cgl@test.com ~]$ find -inum 1315128 -exec head -n 2 {} \;
[cgl@test.com ~]$ lsof | grep deleted                        //查找那些被进程打开，但文件描述符未被释放的进程
sh        19473  cgl    1w      REG    8,3    25896  1315128 /home/cgl/test_log (deleted)
sleep     21647  cgl    1w      REG    8,3    25896  1315128 /home/cgl/test_log (deleted)
```
`结论: test_log文件inode已经被删除了，但是文件描述符还未被释放`

2. 我们使用shell脚本重定向输出到 `test_log` 文件，在运行过程中 `mv` 该文件
```
[cgl@test.com ~]$ sh run.sh > test_log
[cgl@test.com ~]$ ls -i test_log
1315128 test_log
[cgl@test.com ~]$ find -inum 1315128 -exec head -n 2 {} \;   //根据inode查看文件内容
hello cgl ~
hello cgl ~
[cgl@test.com ~]$ mv test_log test_log2
[cgl@test.com ~]$ ls -i test_log2
1315128 test_log2
```
`结论: test_log文件被mv为test_log2，但是inode并不变，证实了上面提到的类Unix操作系统内部不直接使用文件名，而使用 inode 号码来标识文件`   
   
# 三. logrotate
`logrotate`是设计用来管理日志，支持自动滚动切割、压缩、删除以及通过邮件发送日志，支持天级、周级、月级定期处理。 [logrotate man page](https://linux.die.net/man/8/logrotate)

## ① logrotate 安装
1. RHEL/CentOS: yum install logrotate
2. Debian/Ubuntu: apt-get install logrotate
3. Fedora: dnf install logrotate

## ② logrotate 命令行参数
```
[cgl@test.com ~]$ logrotate
logrotate 3.11.0 - Copyright (C) 1995-2001 Red Hat, Inc.
This may be freely redistributed under the terms of the GNU Public License

Usage: logrotate [-dfv?] [-d|--debug] [-f|--force] [-m|--mail=command] [-s|--state=statefile] [-v|--verbose] [-l|--log=STRING]
       [--version] [-?|--help] [--usage] [OPTION...] <configfile>
           
-d,--debug: 打开debug模式，并不会真正改变日志文件
-f,--force: 强制执行文件切割、压缩等操作
-m,--mail <command>: 处理完成后发送邮件
```

## ③ logrotate 配置文件
```
# logrotate log file
# Set "su" directive in config file to tell logrotate which user/group should be used for rotation
su root root
   
/tmp/*.log {        //只处理/tmp目录下所有log结尾日志文件
   compress        //滚动切割后日志使用gizp压缩，nocompress指不压缩
   rotate 2        //日志滚动切割后保留多少个文件
   size 100k       //日志文件超过指定大小后进行滚动切割，支持配置100k、100M、100G
   missingok       //如果日志文件不存在，直接忽略
   notifempty      //如果日志文件为空，则不处理
   copytruncate    //滚动切割动作为: 先拷贝原始日志文件，然后清空原始日志文件
   daily           //每天处理一次，初次之外还可以配置weekly、monthly、yearly等
   olddir /tmp/old //存放历史日志文件目录
   postrotate      //日志滚动切割之后执行后置命令
      echo "done~"
   endscript
   ...
}
```

## ④ logrotate 使用
1. logrotate 安装完成以后默认的配置文件如下
   + /etc/logrotate.conf   //主要的配置文件
   + /etc/logrotate.d/     //logrotate.d是一个目录该目录里的所有文件都会被主动的读入/etc/logrotate.conf中执行
   
   logrotate 是基于`cron`机制来运行的，默认调度配置在 `/etc/cron.daily/logrotate`，每天(6点25)定期执行 `/usr/sbin/logrotate /etc/logrotate.conf`命令。  
   
   `如果只需要每天定期执行，可以通过配置不同配置文件并放到 /etc/logrotate.d/ 目录下，就可以每天被自动执行。`
2. 


