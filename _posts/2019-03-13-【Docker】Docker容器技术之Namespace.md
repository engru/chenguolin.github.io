---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. Namespace 概述
`Namespace`是Linux提供的一种内核级别环境隔离的方法，是Linux创建新进程的一个可选参数。Linux内核实现`Namespace`的目的是为了实现轻量级的虚拟化服务，在同一个Namespace下进程可以彼此感知，但是对外界进程一无所知。这样就可以让容器进程产生错觉，仿佛置身于独立的系统环境之中，所以可以达到隔离的目的。

Linux内核提供了以下7种Namespace类型，具体可以参考[Namespaces in operation](https://lwn.net/Articles/531114/)

| Namespace | 系统调用参数 | 内核版本 | 说明 |
| ------ | ------ | ------ | ------ |
| UTS | CLONE_NEWUTS | Linux 2.6.19 | 主机名和域名的隔离 |
| IPC | CLONE_NEWIPC | Linux 2.6.19 | 进程间通信隔离，信号量、消息队列、共享内存 |
| PID | CLONE_NEWPID | Linux 2.6.24 | PID隔离不同Namespace可以有同一个PID |
| Network | CLONE_NEWNET | Linux 2.6.29 | 网络资源隔离 |
| Mount | CLONE_NEWNS | Linux 2.4.19| 文件系统挂载点隔离 |
| User | CLONE_NEWUSER | Linux 3.8 | 用户和组隔离 |
| Cgroup | CLONE_NEWUSER | Linux 4.6 | Cgroup根目录隔离 |

`Namespace` API包括3个系统调用`clone()`、`unshare()`和`setns()`以及`/proc`目录下的部分文件。在使用这些API的时候，通常需要指定上诉7种Namespace参数的一个或多个，如果是多个使用`|`(位或)操作来实现。

1. `clone()`用于创建独立Namespace子进程，调用方式`int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);`
    * child_func: 子进程运行函数
    * child_stack: 子进程使用的栈空间
    * flags: 使用哪些CLONE_*标志位
    * arg: 用户参数
2. `unshare()`在原先进程上进行Namespace隔离，不启动一个新进程就可以起到隔离效果，相当于跳出原先的Namespace进行操作，调用方式`int unshare(int flags);`
3. `setns()`加入已存在的Namespace中，调用方式`int setns(int fd, int nstype);`
    * fd: 表示我们要加入的Namespace的文件描述符，它是一个指向/proc/$pid/ns目录的文件描述符
    * nstype: 是否要检查fd指向的Namespace类型是否符合我们要求，0表示不检查
    
Linux内核3.8版本之后，用户可以在`/proc/$pid/ns`目录下看到当前进程的Namespace，如下所示4026531839表示Namespace编号。如果两个进程指向的Namespace编号相同，说明它们在同一个Namespace。`/proc/$pid/ns`还有个作用是一旦文件被打开，只要文件描述符存在，即使进程已经结束，Namespace还是会继续存在。
```
$ ls -l /proc/$$/ns         # $$ is replaced by shell's PID
  total 0
  lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
  lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
  lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
  lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
  lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user -> user:[4026531837]
  lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
```

Namespace 虽然看起来隔离了进程，实际上依旧没有完全隔离Linux的资源，比如SELinux、Cgroups以及/sys、/proc/sys、/dev/sd*等目录下的资源。

# 二. UTS Namespace
`UTS`(UNIX Time-sharing System) Namespace提供了`hostname`和`domain name`的隔离，这样每个容器就拥有了独立的主机名和域名。我们可以通过`sethostname()`和`setdomainname()`系统调用来设置，通过`uname()`系统调用获取。

```
/* demo_uts_namespaces.c
 *
 * Copyright 2013, Michael Kerrisk
 * Licensed under GNU General Public License v2 or later
 *
 * Demonstrate the operation of UTS namespaces.
 * */
#define _GNU_SOURCE
#include <sys/wait.h>
#include <sys/utsname.h>
#include <sched.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/* A simple error-handling function: print an error message based
 * on the value in 'errno' and terminate the calling process */

#define errExit(msg)    do { perror(msg); exit(EXIT_FAILURE); \
} while (0)

/* Start function for cloned child */
static int childFunc(void *arg) {
    struct utsname uts;

    /* Change hostname in UTS namespace of child */
    if (sethostname(arg, strlen(arg)) == -1)
	errExit("sethostname");

    /* Retrieve and display hostname */
    if (uname(&uts) == -1)
	errExit("uname");
    
    printf("uts.nodename in child:  %s\n", uts.nodename);

    //char* argv[] = {"ls", "-al", "/", (char*) };
    execv("/bin/sh", NULL);

    /* Terminates child */
    return 0;
}

#define STACK_SIZE (1024 * 1024)    /* Stack size for cloned child */
static char child_stack[STACK_SIZE];

int main(int argc, char *argv[]) {
    pid_t child_pid;
    struct utsname uts;

    if (argc < 2) {
	fprintf(stderr, "Usage: %s <child-hostname>\n", argv[0]);
	exit(EXIT_FAILURE);
    }
	
    /* Display the hostname in parent's UTS namespace. This will be
     * different from the hostname in child's UTS namespace. */
    if (uname(&uts) == -1)
	errExit("uname");
   
    printf("uts.nodename in parent: %s\n", uts.nodename);

    /* Create a child that has its own UTS namespace;
     * the child commences execution in childFunc() */
    child_pid = clone(childFunc, child_stack + STACK_SIZE,
 		CLONE_NEWUTS | SIGCHLD, argv[1]);
    if (child_pid == -1)
	errExit("clone");
    
    printf("PID of child created by clone() is %ld\n", (long) child_pid);

    /* Parent falls through to here */
    /* Give child time to change its hostname */
    if (waitpid(child_pid, NULL, 0) == -1)      /* Wait for child */
	errExit("waitpid");

    printf("child has terminated\n");
    exit(EXIT_SUCCESS);
}
```

1. 编译: gcc uts_namespace.c -o uts_namespace
2. 运行: ./uts_namespace child_process
   ```
   [root@k8s-node-201.95 ns]# ./uts_namespace child_process
   uts.nodename in parent: k8s-node-201.95
   PID of child created by clone() is 12538 
   uts.nodename in child:  child_process
   [root@child_process ns]# hostname
   child_process
   [root@child_process ns]# exit
   exit
   child has terminated
   [root@k8s-node-201.95 ns]#
   ```
3. 从运行结果看，开启UTS Namespace的子进程hostname变成了我们设置的`child_process`，起到了隔离的效果

# 三. IPC Namespace
`IPC`(Interprocess Communication) Namespace是Unix/Linux下进程间通信的一种方式，包括`共享内存`、`信号量`、`消息队列`等方式。IPC隔离能够保证只有同一个Namespace的进程才能相互通信。

1. 首先我们先创建一个消息队列 $ ipcmk -Q
2. 查看已经开启的消息队列 $ ipcs -q   (ipcs是Linux下显示进程间通信设施状态的工具，可以显示消息队列、共享内存和信号量的信息)
   ```
   [root@becwallettest-201 ns]# ipcmk -Q
   消息队列 id：0
   [root@becwallettest-201 ns]# ipcs -q
    --------- 消息队列 -----------
    键        msqid      拥有者  权限     已用字节数  消息
    0xe49db468 0         root   644        0        0
   ```
3. 子进程开启IPC Namespace (核心代码如上UTS Namespace)
   ```
   child_pid = clone(childFunc, child_stack + STACK_SIZE,
	CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, argv[1]);
   ```
4. 编译 gcc ipc_namespace.c -o ipc_namespace
5. 运行
   ```
   [root@k8s-node-201.95 ns]# ./ipc_namespace child_process
   uts.nodename in parent: k8s-node-201.95
   PID of child created by clone() is 14947
   uts.nodename in child:  child_process
   [root@child_process ns]# ipcs -q
    --------- 消息队列 -----------
    键        msqid      拥有者  权限     已用字节数 消息

   [root@child_process ns]# exit
   exit
   child has terminated
   ```
6. 从运行结果看，开启IPC Namespace的子进程，我们运行`ipcs -q`发现已经没有任何消息队列资源，起到了隔离的效果

# 四. PID Namespace
`PID` Namespace用于PID隔离，能够实现不同Namespace下进程拥有相同PID。内核会对所有的PID Namespace维护一个树状结构，顶层是系统初始化时创建的称为root Namespace，新创建的PID Namespace称为child Namespace。这也使得容器中的每个进程有两个PID，容器中的PID和物理机上真实的PID。

1. 每个PID Namespace中第一个进程PID为1，都拥有特权起特殊作用 (类似Linux init进程)。PID为1的进程做为所有进程的父进程，会不断的检查子进程状态，如果有某个子进程称为`孤儿`进程，父进程就会回收资源并结束这个子进程。
2. 在新的PID Namespace中重新挂载/proc目录，使用ps命令才会显示当前Namespace内的进程 (因为ps或top是从宿主机/proc目录获取相关信息，如果没有重新挂载会看到宿主机所有进程)
3. 子进程开启PID Namespace (核心代码如上UTS Namespace)
   ```
   child_pid = clone(childFunc, child_stack + STACK_SIZE,
	CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | SIGCHLD, argv[1]);
   ```
4. 编译 gcc pid_namespace.c -o pid_namespace
5. 运行
   ```
   [root@k8s-node-201.95 ns]# echo $$
   12762
   [root@k8s-node-201.95 ns]# ./pid_namespace  child_process
   uts.nodename in parent: k8s-node-201.95
   PID of child created by clone() is 17399
   uts.nodename in child:  child_process
   [root@child_process ns]# echo $$
   1
   [root@child_process ns]# exit
   exit
   child has terminated
   ```
6. 从运行结果看，开启PID Namespace的子进程，我们在子进程内打印当前shell的PID变成了1，起到了隔离的效果

# 五. Network Namespace
`Network` Namespace提供了网络资源隔离，包括网络设备、IPv4和IPv6协议栈、路由表、防火墙、/proc/net目录、/sys/class/net目录、端口等。

1. 我们可以先在宿主机看下当前的网络接口配置信息 $ ifconfig
   ```
   [root@k8s-node-201.95 ns]# ifconfig
   docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:22:34:5e:4e  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   ens32: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.201.95  netmask 255.255.255.0  broadcast 172.16.201.255
        inet6 fe80::250:56ff:feb8:71d5  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:b8:71:d5  txqueuelen 1000  (Ethernet)
        RX packets 1827555805  bytes 268682195750 (250.2 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 959976055  bytes 196490193474 (182.9 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

   lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 532599411  bytes 216802454389 (201.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 532599411  bytes 216802454389 (201.9 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	
   ```
2. 子进程开启Network Namespace (核心代码如上UTS Namespace)
   ```
   child_pid = clone(childFunc, child_stack + STACK_SIZE,
	CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNET | SIGCHLD, argv[1]);
   ```
3. 编译 gcc net_namespace.c -o net_namespace
5. 运行
   ```
   [root@k8s-node-201.95 ns]# ./net_namespace child_process
    uts.nodename in parent: k8s-node-201.95
    PID of child created by clone() is 20741
    uts.nodename in child:  child_process
   [root@child_process ns]# ifconfig
   [root@child_process ns]# 
   ```
6. 从运行结果看，开启Network Namespace的子进程，我们在子进程使用`ifconfig`命令查找不到任何网络接口配置，起到了隔离的效果

# 六. Mount Namespace
`Mount` Namespace可以用于隔离文件系统挂载点，它是Linux第一个Namespace，隔离后不同Namespace文件系统发生变化互不影响。进程在创建Mount Namespace的时候，会把当前文件结构复制给新的Namespace，新的Namespace中所有的挂载操作都只会影响当前Namespace的文件系统，对外界不会有任何影响。

1. 先查看当前宿主机的/var目录结构
   ```
   [root@k8s-node-201.95 ns]# ls /var
   adm  cache  crash  db  empty  games  gopher  kerberos  lib  local  lock  log  mail  nis  
   opt  preserve  run  spool  tmp  var  www  yp
   ```
2. 子进程开启Mount Namespace (核心代码如上UTS Namespace)
   ```
   child_pid = clone(childFunc, child_stack + STACK_SIZE,
	CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | SIGCHLD, argv[1]);
   ```
3. 子进程重新挂载/var目录
   ```
   // 根目录的挂载类型是 shared，那必须先重新挂载根目录
   mount("", "/", NULL, MS_PRIVATE, "");
   // 重新挂载/var目录
   mount("none", "/var", "tmpfs", 0, "");
   ```
4. 编译 $ gcc mount_namespace.c -o mount_namespace
5. 运行
   ```
   [root@k8s-node-201.95 ns]# gcc mount_namespace.c -o mount_namespace
   [root@k8s-node-201.95 ns]# ./mount_namespace child_process
   uts.nodename in parent: k8s-node-201.95
   PID of child created by clone() is 27972
   uts.nodename in child:  child_process
   [root@child_process ns]# mount | grep none
   none on /var type tmpfs (rw,relatime,seclabel)
   [root@child_process ns]# ls /var
   [root@child_process ns]# exit
   exit
   child has terminated
   [root@k8s-node-201.95 ns]# mount | grep none
   [root@k8s-node-201.95 ns]# ls /var
   adm  cache  crash  db  empty  games  gopher  kerberos  lib  local  lock  log  mail  nis  
   opt  preserve  run  spool  tmp  var  www  yp
   ```
6. 从运行结果看，开启Mount Namespace的子进程，我们在子进程内的mount操作把/var目录给隔离了，退出子进程后自动unmount不影响宿主机，起到了隔离的效果
7. Docker容器底层实现原理是进程使用Mount Namespace参数，在容器进程启动之前通过`pivot_root`或`chroot`切换容器根目录到rootfs挂载点(/var/lib/docker/aufs/mnt或/var/lib/docker/devicemapper/mnt)，使得容器进程文件系统和外界隔离，在容器内进行各种操作并不会影响宿主机，起到了隔离效果。

# 七. User Namespace
`User` Namespace为正在运行的进程提供安全相关的隔离，包括用户ID、用户组ID、root目录、key以及特殊权限，一个普通用户进程通过clone()创建新的子进程中设置User Namespace可以拥有不同的用户和用户组。我们通过User Namespace把宿主机中的一个普通用户(只有普通权限的用户)映射到容器中的root用户。在容器中，该用户认为自己就是root，也具有root的各种权限，但是对于宿主机上的资源，它只有很有限的访问权限(普通用户)。

Docker默认没有隔离宿主机用户和容器用户，docker已经实现了相关功能，只是没有开启。

1. 子进程开启User Namespace (User namespace 是 Linux 3.8 新增的一种Namespace)
   ```
   child_pid = clone(childFunc, child_stack + STACK_SIZE,
	CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUSER | SIGCHLD, argv[1]);
   ```
2. 使用了这个User Namespace参数后，子进程内部看到的UID和GID已经与外部不同了，默认显示为65534。因为子进程找不到其真正的UID所以，设置上了最大的UID（其设置定义在/proc/sys/kernel/overflowuid）
3. 要把子进程内UID与宿主机上UID进行映射，需要修改`/proc/$pid/uid_map`和`/pro/$pid/gid_map`这2个文件  
   文件格式为`ID-inside-ns ID-outside-ns length`
    * ID-inside-ns: 表示在子进程显示的UID或GID
    * ID-outside-ns: 表示宿主机真实的UID或GID
    * length: 表示映射的范围，一般填1，表示一一对应

   例如宿主机的uid=1000映射成子进程内的uid=0  
   $ cat /proc/$pid/uid_map  
     0       1000          1
4. 子进程中设置uid_map和gid_map两个文件
   ```
   void set_uid_map(pid_t pid, int inside_id, int outside_id, int length) {
       char path[256];
       sprintf(path, "/proc/%d/uid_map", getpid());
       FILE* uid_map = fopen(path, "w");
       fprintf(uid_map, "%d %d %d", inside_id, outside_id, length);
       fclose(uid_map);
   }
   
   void set_gid_map(pid_t pid, int inside_id, int outside_id, int length) {
       char path[256];
       sprintf(path, "/proc/%d/gid_map", getpid());
       FILE* gid_map = fopen(path, "w");
       fprintf(gid_map, "%d %d %d", inside_id, outside_id, length);
       fclose(gid_map);
   }

   static int childFunc(void *arg) {
       struct utsname uts;

       /* Change hostname in UTS namespace of child */
       if (sethostname(arg, strlen(arg)) == -1)
	   errExit("sethostname");

       /* Retrieve and display hostname */
       if (uname(&uts) == -1)
	   errExit("uname");

       printf("uts.nodename in child:  %s\n", uts.nodename);

       // 设置uid map
       set_uid_map(getpid(), 0, 1000, 1);
       // 设置gid map
       set_gid_map(getpid(), 0, 1000, 1);

       printf("eUID = %ld;  eGID = %ld;  ", (long) geteuid(), (long) getegid());

       cap_t caps;
       caps = cap_get_proc();
       printf("capabilities: %s\n", cap_to_text(caps, NULL));

       //char* argv[] = {"ls", "-al", "/", (char*) };
       execv("/bin/sh", NULL);

       /* Terminates child */
       return 0;
   }
   ```
5. 编译 $ gcc user_namespace.c -o user_namespace
6. 运行
   ```
   [root@k8s-node-201.95 ns]# ./user_namespace child_process
   uts.nodename in parent: k8s-node-201.95
   PID of child created by clone() is 27972
   uts.nodename in child:  child_process
   eUID = 0;  eGID = 0;  capabilities: = [...],37+ep
   [root@child_process ns]# exit
   exit
   child has terminated
   [root@k8s-node-201.95 ns]
   ```
7. 虽然容器内是root用户，实际上在宿主机还是以普通用户身份运行容器进程，因此容器安全性能够得到保证，同时也给容器极大的自由。

# 八. Docker 使用Namespace
Docker创建一个容器时，它会创建新的以上几种 Namespace 的实例，然后把容器中的所有进程放到这些 Namespace 之中，使得Docker容器中的进程只能看到隔离的系统资源。 

Docker run 命令有以下4个参数和 Namespace 相关  
   + --ipc string IPC namespace to use
   + --pid string PID namespace to use
   + --userns string User namespace to use
   + --uts string UTS namespace to use

1. 创建容器  
   `$ docker run -it --memory 100M --cpu-period=100000 --cpu-quota=50000 --cpuset-cpus 1 --device /dev/sda:/dev/sda --device-read-bps /dev/sda:1mB --device-write-bps /dev/sda:1mB ubuntu`
   
   容器id为`13a918bb87f0482d5fddd8c9ad43521ae1d9e499d93b032d5e15ff1ccad3153f`
   
   + --memory 100M: 限制最大内存是使用为100M
   + --cpu-period=100000: 设置容器进程每个周期CPU总时间为100ms
   + --cpu-quota=50000: 设置容器进程能够使用CPU时间为50ms，即设置使用0.5个CPU
   + --cpuset-cpus 1: 限制容器进程只使用第一个CPU核
   + --device-read-bps /dev/sda:1mB: 限制设备读取速度为1MB/s
   + --device-write-bps /dev/sda:1mB: 限制设备写速度为1MB/s
   
2. 查看容器在物理机PID，找到PID为36992  
   `$ docker inspect 13a918bb87f0482d5fddd8c9ad43521ae1d9e499d93b032d5e15ff1ccad3153f | grep Pid`

3. 查看 36992 进程Namespace情况
   `$ cd /proc/36992/ns`
   ```
   [root@localhost ns]# ls -lsrt
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:52 net -> net:[4026533062]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 uts -> uts:[4026533058]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 user -> user:[4026531837]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 pid_for_children -> pid:[4026533060]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 pid -> pid:[4026533060]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 mnt -> mnt:[4026533057]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 ipc -> ipc:[4026533059]
   0 lrwxrwxrwx. 1 root root 0 5月  11 20:54 cgroup -> cgroup:[4026531835]
   ```
   
