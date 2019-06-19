---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Spark
---


# 简介
  MapReduct框架从hadoop-0.23版本以后发生了重大的变革, 新的计算框架我们称为MapReudce 2.0 或者 YARN(Yet-Another-Resource-Negotiator)
  
  这篇文章主要是简单的介绍YARN的架构, 因为后续我们都会使用YARN提交Spark程序
  
  MapReduct 2.0最主要的变革是把资源分配和任务调度隔离. 一个全局的资源管理称为ResourceManager (RM), 同时每个应用本身有一个ApplicationMaster (AM). 应用可以是单独的Job也可用是MapReduce job或者是是一个DAG job.

  ResourceManager和NodeManager (NM)组成数据处理框架, ResourceManager对所有应用进行资源分配,管理和调度。每个应用程序都有一个ApplicationMaster, ApplicationMaster从ResourceManager分配到资源, 在NodeManager执行和监控任务

# 架构
  ![](https://camo.githubusercontent.com/75c2e6c15e58006fa3f936170d4f5172259d5681/68747470733a2f2f6861646f6f702e6170616368652e6f72672f646f63732f72322e342e312f6861646f6f702d7961726e2f6861646f6f702d7961726e2d736974652f7961726e5f6172636869746563747572652e676966)
  ![](https://camo.githubusercontent.com/57eb234be536b1b6fb5e9b3c7a7aa9fa636a5091/687474703a2f2f7777772e69626d2e636f6d2f646576656c6f706572776f726b732f636e2f646174612f6c6962726172792f62642d7961726e2d696e74726f2f466967757265334172636869746563747572652d6f662d5941524e2e706e67)

1. 在 YARN 架构中，一个全局 ResourceManager 以主要后台进程的形式运行，它通常在专用机器上运行，在各种竞争的应用程序之间仲裁可用的集群资源。
2. ResourceManager 会追踪集群中有多少可用的活动节点和资源，协调用户提交的哪些应用程序应该在何时获取这些资源。ResourceManager 是惟一拥有此信息的进程，所以它可通过某种共享的、安全的、多租户的方式制定分配（或者调度）决策（例如，依据应用程序优先级、队列容量、ACLs、数据位置等）。在用户提交一个应用程序时，一个称为 ApplicationMaster 的轻量型进程实例会启动来协调应用程序内的所有任务的执行。这包括监视任务，重新启动失败的任务，推测性地运行缓慢的任务，以及计算应用程序计数器值的总和。这些职责以前分配给所有作业的单个 JobTracker。ApplicationMaster 和属于它的应用程序的任务，在受 NodeManager 控制的资源容器中运行。
3. NodeManager 是 TaskTracker 的一种更加普通和高效的版本。没有固定数量的 map 和 reduce slots，NodeManager 拥有许多动态创建的资源容器。容器的大小取决于它所包含的资源量，比如内存、CPU、磁盘和网络 IO。目前，仅支持内存和 CPU (YARN-3)。未来可使用 cgroups 来控制磁盘和网络IO。一个节点上的容器数量，由配置参数与专用于从属后台进程和操作系统的资源以外的节点资源总量（比如总 CPU 数和总内存）共同决定。
4. ApplicationMaster 可在容器内运行任何类型的任务。例如，MapReduce ApplicationMaster 请求一个容器来启动 map 或 reduce 任务，而 Giraph ApplicationMaster 请求一个容器来运行 Giraph 任务。您还可以实现一个自定义的 ApplicationMaster 来运行特定的任务，进而发明出一种全新的分布式应用程序框架，改变大数据世界的格局。

在 YARN 中，MapReduce 降级为一个分布式应用程序的一个角色（但仍是一个非常流行且有用的角色），现在称为 MRv2。MRv2 是经典 MapReduce 引擎（现在称为 MRv1）的重现，运行在 YARN 之上。
