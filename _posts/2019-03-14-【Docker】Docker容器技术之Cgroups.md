---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

# 一. Cgroups 概述
`Cgroups`全称是Linux Control Groups，它是Linux内核提供的一种机制，用于`限制、记录、隔离进程组所使用的物理资源（CPU、Memory、IO）`。2006年的时候由Google的工程师提出，最初的名字叫`Process Containers`，2007年的时候改名为`Control Groups`，并合并到2008年发布的2.6.24版本内核。

Cgroups设计目的就是为进程提供资源控制，主要包括以下4个功能
1. `资源限制（Resource Limitation）`: 控制进程组使用资源上限，例如最大内存、最多CPU时间等等
2. `优先级控制（Prioritization）`: 通过分配CPU时间片数量以及硬盘IO大小就相当于是设置进程优先级
3. `资源统计（Accounting）`: 统计系统资源使用量，例如CPU使用时长、内存使用量
4. `进程控制（Control）`: 对进程组执行挂起、恢复等操作

Cgroups有几个非常重要的概念
1. `task（任务）`: task表示系统中的一个进程
2. `cgroup（控制组）`: cgroup表示按某种资源控制标准划分而成的进程组，可以绑定多个子资源
3. `subsystem（子资源系统）`: subsystem表示资源调度控制器，例如cpu、memory等
4. `hierarchy（层级树）`: hierarchy由一系列cgroup以一个树状结构排列而成，每个hierarchy通过绑定对应的subsystem进行资源调度

在Linux中Cgroups给用户暴露出来的接口是文件系统，它是以文件和目录的形式组织在操作系统的`/sys/fs/cgroup`路径下的，使用`mount -t cgroup`命令可以查看Cgroups的挂载目录，可以看到`/sys/fs/cgroup`目录下有很多子目录例如`cpu、cpuset、memory`，这些都是当前可以被Cgroups限制的资源种类。

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

Cgroups子资源系统实际上就是资源控制器，每种子资源系统控制一种资源，包括以下几种
1. `blkio`: 限制块设备的IO速率（磁盘、SSD、USB等）
2. `cpu`: 限制task能够使用的CPU时间
3. `cpuacct`: 自动生成task使用资源统计报告
4. `cpuset`: 限制task能够运行在哪些CPU核上
5. `devices`: 限制task对设备的访问，允许或拒绝
6. `freezer`: 挂起或重启task
7. `memory`: 限制task能够使用的内存上限
8. `net_cls`: 通过标记网络数据包，从而允许Linux流量控制程序（TC：Traffic Controller）识别从具体cgroup中生成的数据包
9. `perf_event`: 用来做性能分析

`Cgroups`的实现本质上是给系统进程挂上钩子（hooks），当task运行的过程中涉及到某个资源时就会触发钩子上所附带的subsystem进行检测，最终根据资源类别的不同使用对应的技术进行资源限制和优先级分配。

# 二. Cgroups subsystem
`/sys/fs/cgroup/`不同子资源系统下会有许多配置文件，不同子资源系统对应有不同文件，但是以下4个文件是所有子资源系统都会有的。

1. `tasks`: 当前控制组包含的PID列表，把某个PID添加到tasks文件相当于把进程加入该控制组
2. `cgroup.procs`: 当前控制组包含的线程组ID，把某个线程组ID添加到cgroup.procs文件相当于把这个组所有线程都加入到当前控制组
3. `notify_on_release`: 0或1，是否在控制组销毁的时候执行release_agent命令，默认为0表示不执行
4. `release_agent`: 要执行的命令

## ① memory
`memory`子资源系统用来限制进程能够使用的内存上限，也能统计内存使用情况

1. 创建一个控制组名为`cgltest`
   ```
   $ cd /sys/fs/cgroup/memory
   $ mkdir cgltest
   ```
2. 查看控制组内容
   ```
   $ cd cgltest
   $ ls
   cgroup.clone_children        memory.kmem.failcnt              memory.kmem.tcp.limit_in_bytes      
   memory.max_usage_in_bytes    memory.move_charge_at_immigrate  memory.stat     tasks
   cgroup.event_control         memory.kmem.limit_in_bytes       memory.kmem.tcp.max_usage_in_bytes  
   memory.memsw.failcnt         memory.numa_stat                 memory.swappiness
   cgroup.procs                 memory.kmem.max_usage_in_bytes   memory.kmem.tcp.usage_in_bytes      
   memory.memsw.limit_in_bytes  memory.oom_control               memory.usage_in_bytes
   memory.failcnt               memory.kmem.slabinfo             memory.kmem.usage_in_bytes         
   memory.memsw.max_usage_in_bytes    memory.pressure_level      memory.use_hierarchy
   memory.force_empty           memory.kmem.tcp.failcnt          memory.limit_in_bytes               
   memory.memsw.usage_in_bytes  memory.soft_limit_in_bytes       notify_on_release
   ```
   在memory子资源系统下创建一个控制组默认会生成许多配置文件
3. 配置文件详解
    * 控制内存配置
        * memory.limit_in_bytes: 进程能够使用的内存上限，单位为字节，也可以添加k/K、m/M和g/G单位后缀，-1表示不限制
        * memory.memsw.limit_in_bytes: 进程能够使用内存加交换分区的上限，单位为字节，也可以添加k/K、m/M和g/G单位后缀，-1表示不限制
        * memory.soft_limit_in_bytes: 进程使用内存软上限，当内存充足的时候允许进程使用到memory.limit_in_bytes设定只，当内存不足的时候限制进程内存使用不超过memory.soft_limit_in_bytes设定值
        * memory.swappiness: 控制内核使用交换分区的倾向，取值范围是0~100之间的整数(包含0和100)，值越小越倾向使用物理内存
        * memory.kmem.limit_in_bytes: 控制内核使用内存上限
        * memory.oom_control: 是否启动oom killer，默认值为0表示oom后kill掉进程，值为1表示oom后不杀死进程而是暂停进程，直到它释放了内存
    * 内存统计配置
        * memory.failcnt: 进程内存使用超过memory.limit_in_bytes次数
        * memory.memsw.failcnt: 进程内存加上交换分区使用超过memory.memsw.limit_in_bytes次数
        * memory.stat: 内存使用情况统计
        * memory.usage_in_bytes: 当前使用的总内存字节数
        * memory.memsw.usage_in_bytes: 当前使用的总内存加交换分区字节数
        * memory.max_usage_in_bytes: 进程使用的最大内存字节数
        * memory.memsw.max_usage_in_bytes: 进程使用的最大内存加交换分区字节数
4. docker参数
    * -m, --memory bytes: 进程能够使用的内存上限，对应memory.limit_in_bytes文件
    * --memory-swap bytes: 进程能够使用内存加交换分区的上限，对应memory.memsw.limit_in_bytes文件
    * --memory-reservation bytes: 进程使用内存软上限，对应memory.soft_limit_in_bytes文件
    * --memory-swappiness int: 进程内核使用交换分区的倾向，对应memory.swappiness文件
    * --kernel-memory bytes: 内核使用内存上限，对应memory.kmem.limit_in_bytes文件
    * --oom-kill-disable: 是否启动oom killer，对应memory.oom_control文件
5. 本地测试  
   1). 设置memory.limit_in_bytes为10M  
   2). 设置memory.memsw.limit_in_bytes为20M  
   3). 本地运行一下程序 (故意内存泄露)  
      ```
      #include<stdio.h>
      #include<malloc.h>
      int main() {
          while (1)  {
             char *p;
             p = (char *)malloc(100);
          }
          return 0;
      }
      ```
      编译: $ gcc main.c -o main  
      运行: $ ./main &      //得到进程PID为38610  
   4). 查看进程内存使用  
      ```
      $ top -p 38610
      PID   USER  PR  NI   VIRT    RES   SHR S  %CPU  %MEM   TIME+
      40984 root  20  0   6806972  6.5g  892 R  100.0 20.7   0:07.55
      ```
      发现内存直接使用到了6.5g  
   5). 把PID 38610加入tasks文件  
      $ echo 38610 > /sys/fs/cgroup/memory/cgltest/tasks  
   6). 进程被kill，我们可以通过dmesg查看oom信息
      ```
      [4814551.793590] main invoked oom-killer: gfp_mask=0x14000c0(GFP_KERNEL), nodemask=(null), order=0, oom_score_adj=0
      [4814551.793593] main cpuset=/ mems_allowed=0-1
      [4814551.793602] CPU: 7 PID: 38610 Comm: main Not tainted 4.15.7-1.el7.elrepo.x86_64 #1
      [4814551.793604] Hardware name: Dell Inc. PowerEdge R420/0VD50G, BIOS 2.1.2 01/20/2014
      [4814551.793606] Call Trace:
      ...
      [4814551.793726] kmem: usage 492kB, limit 9007199254740988kB, failcnt 0
      [4814551.793727] Memory cgroup stats for /cgltest: cache:0KB rss:9748KB rss_huge:0KB shmem:0KB mapped_file:0KB dirty:0KB writeback:0KB swap:0KB inactive_anon:4992KB active_anon:4756KB inactive_file:0KB active_file:0KB unevictable:0KB
      [4814551.793757] [ pid ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
      [4814551.794082] [38610]     0 38610  2219381  2218486 17838080        0             0 main
      [4814551.794087] Memory cgroup out of memory: Kill process 38610 (main) score 842248 or sacrifice child
      [4814551.795972] Killed process 38610 (main) total-vm:8877524kB, anon-rss:8873096kB, file-rss:848kB, shmem-rss:0kB
      [4814552.597177] oom_reaper: reaped process 38610 (main), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
      ```
        
## ② cpu
`cpu`子资源系统用来限制进程能够使用的CPU上限，也能统计CPU使用情况

目前Linux内核主要有两大类调度算法`完全公平调度算法(CFS：Completely Fair Scheduler)`和`实时调度算法(Real-Time Scheduler)`，不同调度策略会有不同配置文件，Linux默认为CFS调度算法

1. 创建一个控制组名为`cgltest`
   ```
   $ cd /sys/fs/cgroup/cpu
   $ mkdir cgltest
   ```
2. 查看控制组内容
   ```
   $ cd cgltest
   $ ls
   cgroup.clone_children      cpuacct.usage_percpu_sys  cpu.rt_period_us cgroup.procs    
   cpuacct.usage_percpu_user  cpu.rt_runtime_us         cpuacct.stat               
   cpuacct.usage_sys          cpu.shares                cpuacct.usage          
   cpuacct.usage_user         cpu.stat                  cpuacct.usage_all      
   cpu.cfs_period_us          notify_on_release         cpuacct.usage_percpu            
   cpu.cfs_quota_us           tasks
   ```
3. 配置文件详解
    * CFS调度配置 (Linux内核默认CFS调度)
        * cpu.shares: 进程使用CPU的权重设置，默认值为1024。当CPU资源充足时，设置CPU的权重是没有意义的，只有CPU资源紧张的情况下，CPU的权重才能让不同的进程按比例分到不同的CPU使用时长
        * cpu.cfs_period_us: 每个周期CPU总时间，配合cpu.cfs_quota_us使用，docker中默认设置为100000(100ms)
        * cpu.cfs_quota_us: 进程每个周期能使用CPU时间，配合cpu.cfs_period_us使用，默认-1表示不限制。例如要限制进程只能使用50% CPU那可以设置该值为cpu.cfs_period_us的1/2，如果要设置进程能使用多核CPU那可以设置该值为cpu.cfs_period_us的n倍
    * RT调度配置 (只限制实时任务的CPU)
        * cpu.rt_period_us: 每个周期总使用CPU时间
        * cpu.rt_runtime_us: 每个周期进程能够使用CPU时间
    * 内存统计配置
        * cpu.stat: CPU使用统计数据
            * nr_periods: 总共经历了几个CPU周期
            * nr_throttled: 进程被限制使用CPU的次数
            * throttled_time: 进程被限制的总时间
        * cpuacct.usage: 所有进程总共使用CPU时间统计
        * cpuacct.stat: 所有进程按用户态、内核态区分总共使用CPU时间统计
        * cpuacct.usage_percpu: 所有进程各个CPU使用时间统计
4. docker参数
    * --cpu-period int: CFS调度 设置每个周期CPU总时间，默认为100000(100ms)，对应cpu.cfs_period_us文件
    * --cpu-quota int: CFS调度 设置进程每个周期能使用CPU时间，默认-1表示不限制，对应cpu.cfs_quota_us文件
    * --cpu-rt-period int: RT调度 设置每个周期总使用CPU时间，对应cpu.rt_period_us文件
    * --cpu-rt-runtime int: RT调度 每个周期进程能够使用CPU时间，对应cpu.rt_runtime_us文件
    * -c, --cpu-shares int: 设置进程使用CPU的权重，对应cpu.shares文件
    * --cpus decimal: 设置使用CPU个数，支持小数点。实际上是--cpu-period和--cpu-quota两个参数的封装
5. 本地测试  
   1). 设置cpu.cfs_period_us为100000  
   2). 设置cpu.cfs_quota_us为50000 (期望使用0.5个CPU)  
   3). 本地运行一下程序  
      ```
       #include<stdio.h>
       #include<malloc.h>
       int main() {
           while (1)  {
           }
           return 0;
       }
      ```
      编译: $ gcc main.c -o main  
      运行: $ ./main &      //得到进程PID为48091  
   4). 查看进程CPU使用
      ```
       $ top -p 48091
       PID   USER  PR   NI  VIRT  RES  SHR  S %CPU  %MEM   TIME+
       48091 root  20   0   4220  640  564  R 100.0  0.0   0:40.06
      ```
      发现CPU直接使用到了100%  
   5). 把PID 48091加入tasks文件  
       $ echo 48091 > /sys/fs/cgroup/cpu/cgltest/tasks  
   6). 进程CPU使用率降到了50%，符合我们设置的  
      ```
       PID   USER  PR  NI  VIRT  RES  SHR S  %CPU %MEM  TIME+
       48091 root  20  0   4220  640  564 R  50.0  0.0  3:04.26
      ```

## ③ cpuset
`cpu`子资源系统用来限制进程能够使用哪些特定CPU核上，以及哪些特定内存节点

1. 创建一个控制组名为`cgltest`
   ```
   $ cd /sys/fs/cgroup/cpuset
   $ mkdir cgltest
   ```
2. 查看控制组内容
   ```
   $ cd cgltest
   $ ls
   cgroup.clone_children      cpuset.memory_pressure  cgroup.procs           
   cpuset.memory_spread_page  cpuset.cpu_exclusive    cpuset.memory_spread_slab
   cpuset.cpus                cpuset.mems             cpuset.effective_cpus  
   cpuset.sched_load_balance  cpuset.effective_mems   cpuset.sched_relax_domain_level
   cpuset.mem_exclusive       notify_on_release       cpuset.mem_hardwall    
   tasks                      cpuset.memory_migrate
   ```
3. 配置文件详解
    * cpuset.cpus: 设置进程能够使用哪些特定的CPU，内容使用逗号分隔列表，例如0-2,7 表示CPU第0，1，2，和7核，默认为空表示使用所有
    * cpuset.mems: 设置进程能够使用哪些特定的内存节点，使用逗号分隔列表，默认为空表示使用所有
4. docker参数
    * --cpuset-cpus string: 设置进程能够使用哪些特定的CPU，对应cpuset.cpus文件
    * --cpuset-mems string: 设置进程能够使用哪些特定的内存节点，对应cpuset.mems文件

## ④ blkio
`blkio`子资源系统用来限制块设备的IO速率（磁盘、SSD、USB等）

限制块设备IO目前有2种策略，一种是基于完全公平队列调度(CFQ：Completely Fair Queuing)的按权重分配各个控制组所能占用总体资源的百分比，另一种是设定资源使用上限

1. 创建一个控制组名为`cgltest`
   ```
   $ cd /sys/fs/cgroup/blkio
   $ mkdir cgltest
   ```
2. 查看控制组内容
   ```
   $ cd cgltest
   $ ls
   blkio.io_merged                  blkio.sectors_recursive        blkio.io_merged_recursive
   blkio.throttle.io_service_bytes  blkio.io_queued                blkio.throttle.io_serviced
   blkio.io_queued_recursive        blkio.throttle.read_bps_device blkio.io_service_bytes             
   blkio.throttle.read_iops_device  blkio.io_service_bytes_recursive  blkio.throttle.write_bps_device
   blkio.io_serviced                blkio.throttle.write_iops_device  blkio.io_serviced_recursive       
   blkio.time                       blkio.io_service_time             blkio.time_recursive
   blkio.io_service_time_recursive  blkio.weight                      blkio.io_wait_time         
   blkio.weight_device              blkio.io_wait_time_recursive      cgroup.clone_children
   blkio.leaf_weight                cgroup.procs                      blkio.leaf_weight_device    
   notify_on_release                blkio.reset_stats                 tasks
   blkio.sectors
   ```
3. 配置文件详解
    * 设置权重
        * blkio.weight: 设置块设备读写权重，值范围在100~1000
        * blkio.weight_device: 针对特定设备设置读写权重，格式为device_types:node_numbers weight
    * 设置上限
        * blkio.throttle.read_bps_device: 每秒钟最多读取多少字节，默认为空表示不限制，内容格式为device_types:node_numbers bytes_per_second
        * blkio.throttle.read_iops_device: 每秒在最多读取次，默认为空表示不限制，内容格式为device_types:node_numbers operations_per_second
        * blkio.throttle.write_bps_device: 每秒钟最多写多少字节，默认为空表示不限制，内容格式为device_types:node_numbers bytes_per_second
        * blkio.throttle.write_iops_device: 每秒钟最多写多少次，默认为空表示不限制，内容格式为device_types:node_numbers operations_per_second
    * 统计数据
        * blkio.throttle.io_serviced: 进程读写块设备次数统计
        * blkio.throttle.io_service_bytes: 进程读写块设备字节数统计
        * blkio.time: 对各个设备的访问时间统计
        * blkio.io_serviced: CFQ调度下进程读写块设备次数统计，与blkio.throttle.io_serviced相反
        * blkio.io_services_bytes: CFQ调度下进程读写块设备字节数统计，与blkio.throttle.io_service_bytes相反
        * blkio.io_queued: 队列中IO操作次数统计
        * blkio.io_wait_time: 各设备各种IO类型在队列中等待时间统计
        * blkio.io_service_time: 特定设备IO操作实际
4. docker参数
    * --blkio-weight uint16: 设置块设备读写权重，对应blkio.weight文件
    * --blkio-weight-device list: 设置特定设备设置读写权重，对应blkio.weight_device文件
    * --device-read-bps list: 设置每秒钟最多读取多少字节，对应blkio.throttle.read_bps_device文件
    * --device-read-iops list: 设置每秒在最多读取次，对应blkio.throttle.read_iops_device文件
    * --device-write-bps list: 设置每秒钟最多写多少字节，对应blkio.throttle.write_bps_device文件
    * --device-write-iops list: 设置每秒钟最多写多少次，对应blkio.throttle.write_iops_device文件
    
## ⑤ device
`device`子资源系统用来限制进程允许或限制访问某些设备

1. 创建一个控制组名为`cgltest`
   ```
   $ cd /sys/fs/cgroup/device
   $ mkdir cgltest
   ```
2. 查看控制组内容
   ```
   $ cd cgltest
   $ ls
   cgroup.clone_children  cgroup.procs  devices.allow  
   devices.deny  devices.list  notify_on_release  tasks
   ```
3. 配置文件详解
    * devices.allow: 进程能够访问的设备列表，内容格式为`type device_types:node_numbers access`
        * type: 表示类型，有3种类型 a(all), c(char), b(block)
        * major:minor: 表示设备编号，*:* 表示所有设备
        * access: 访问方式，有3种类型 r(read), w(write), m(mknod) 
    * devices.deny: 进程禁止访问的设备列表，内容格式同上
    * devices.list: 当前控制组设置的访问控制设备列表，默认为 `a *:* rwm` 表示所有设备

## ⑥ freezer
`freezer`用于恢复/暂停进程

1. 创建一个控制组名为`cgltest`
   ```
   $ cd /sys/fs/cgroup/freezer
   $ mkdir cgltest
   ```
2. 查看控制组内容
   ```
   $ cd cgltest
   $ ls
   cgroup.clone_children  cgroup.procs  freezer.parent_freezing  
   freezer.self_freezing  freezer.state  notify_on_release  tasks
   ```
3. 配置文件详解
    * freezer.state: 进程的状态，有3种状态
        * FROZEN: 进程都被挂起（暂停）
        * FREEZING: 进程正在被挂起的过程中
        * THAWED: 进程已经正常恢复
4. 如果要挂起一个进程，可以把进程PID写入`tasks`文件，然后修改`freezer.state`文件为`FROZEN`
5. 本地测试  
    1). 本地运行以下程序  
      ```
       #include<stdio.h>
       #include<malloc.h>
       int main() {
           while (1)  {
               printf("hello world \n");
           }
           return 0;
       }
      ```
      编译: $ gcc main.c -o main  
      运行: $ ./main &      //得到进程PID为 6340  
   2). top查看进程信息  
      ```
       $ top -p 6340
       PID   USER  PR   NI  VIRT  RES  SHR  S %CPU  %MEM   TIME+
       6340  root  20   0   4220  620  544  R 100.0  0.0   0:11.71
      ```
   3). 挂起进程  
      $ echo 6340 > /sys/fs/cgroup/freezer/cgltest/tasks  
      $ echo FROZEN > /sys/fs/cgroup/freezer/cgltest/freezer.state  
   4). 再次查看top发现CPU使用率降到了0%，同时进程不再输出，进程被挂起  
      ```
       PID  USER  PR  NI  VIRT  RES  SHR S  %CPU %MEM   TIME+
       6340 root  20  0   4220  620  544 D   0.0  0.0   1:22.41
      ```
   5). 恢复进程  
      $ echo THAWED > /sys/fs/cgroup/freezer/cgltest/freezer.state  
   6). 再次查看top发现CPU使用率又回到了100%，继续打印输出，进程被恢复  
    
# 三. Docker 使用Cgroups
默认情况下操作系统已经把Cgroups挂载到文件系统上`/sys/fs/cgroup`目录上，通过`mount -t cgroup`命令即可查看。所以，如果要使用Cgroups限制进程组资源使用，我们可以直接操作/sys/fs/cgroup目录下对应子目录。

Docker使用Cgroups本质是`docker run`运行一个新容器的时候在`/sys/fs/cgroup`每个子目录下创建一个`docker/[container_id]`子目录，把启动配置参数配置在对应子目录下（cpu、memory等），然后把容器进程`PID`写入`tasks`文件，最终起到资源限制的作用。

1. 创建容器  
   `$ docker run -it --memory 100M --cpu-period=100000 --cpu-quota=50000 --cpuset-cpus 1 --device /dev/sda:/dev/sda --device-read-bps /dev/sda:1mB --device-write-bps /dev/sda:1mB ubuntu`
   
   容器id为`4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e`
   
   + --memory 100M: 限制最大内存是使用为100M
   + --cpu-period=100000: 设置容器进程每个周期CPU总时间为100ms
   + --cpu-quota=50000: 设置容器进程能够使用CPU时间为50ms，即设置使用0.5个CPU
   + --cpuset-cpus 1: 限制容器进程只使用第一个CPU核
   + --device-read-bps /dev/sda:1mB: 限制设备读取速度为1MB/s
   + --device-write-bps /dev/sda:1mB: 限制设备写速度为1MB/s
   
2. 查看Cgroups memory资源限制配置
   ```
   $ cat /sys/fs/cgroup/memory/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/memory.limit_in_bytes
     104857600 //最大内存使用为100M  
    
   $ cat /sys/fs/cgroup/memory/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/tasks   
     18475     //容器进程pid
   ```
3. 查看Cgroups cpu资源限制配置
   ```
   $ cat /sys/fs/cgroup/cpu/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/cpu.cfs_period_us  
     100000    //容器进程每个周期CPU总时间为100ms
    
   $ cat /sys/fs/cgroup/cpu/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/cpu.cfs_quota_us  
     50000     //容器进程能够使用CPU时间为50ms
     
   $ cat /sys/fs/cgroup/cpu/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/tasks   
     18475    //容器进程pid
   ```
4. 查看Cgroups cpuset资源限制配置
   ```
   $ cat /sys/fs/cgroup/cpuset/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/cpu.cpus  
     1        //容器进程只使用第一个CPU核
   
   $ cat /sys/fs/cgroup/cpuset/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/tasks   
     18475    //容器进程pid
   ```
5. 查看Cgroups blkio资源限制配置
   ```
   $ cat /sys/fs/cgroup/blkio/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/blkio.throttle.read_bps_device  
     8:0 1048576   //限制/dev/sda设备读取速度为1MB/s
   
   $ cat /sys/fs/cgroup/blkio/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/blkio.throttle.write_bps_device  
     8:0 1048576   //限制/dev/sda设备写速度为1MB/s
     
   $ cat /sys/fs/cgroup/blkio/docker/4bd8e5259cff657034e7fcfb2232454e407641d4f5e7b34999c78f3bb557624e/tasks   
     18475    //容器进程pid
   ```
