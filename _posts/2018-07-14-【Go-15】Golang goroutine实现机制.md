---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Go
---

# 一. 操作系统调度
## ① 并行与并发

## 

# 二. Golang调度
## ① Goroutine定义


## ② Goroutine和线程的区别
1. `内存消耗`: 创建一个goroutine比创建一个线程需要的内存更少，创建一个线程需要`1MB`的内存空间，而创建一个goroutine只需要`2KB`的内存空间，比线程节省了500倍，在Golang程序中一个线程可以承载上千个goroutine。
2. `启动和销毁的成本`: 线程的启动和销毁必须要和操作系统内核进行交互，goroutine的启动和销毁都是通过Go runtime，Go runtime负责管理调度、垃圾回收、goroutine创建等，goroutine相对于线程来说启动和销毁成本低很多。
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-vs-thread.png?raw=true)
3. `上下文切换成本`: goroutine和线程最大的区别是上下文切换成本，线程是抢占式调度的，当线程运行CPU时间片到了之后就会被另外一个可运行线程抢占，这个过程需要涉及到上下文切换，线程需要保存所有的寄存器，包括通用寄存器、程序计数器等。goroutine调度是协作式的，goroutine的调度不需要和操作系统内核进行交互，当goroutine进行切换的时候只需要保存少量的程序计数器，成本比线程低很多。

## ③ Goroutine调度本质
1. Go使用以下3个实体描述goroutine调度，通过这3个Go就可以实现`M:N`的调度，也就是M个goroutine可以运行在N个系统线程上，任何时候每个系统线程 M 都可以执行一个G，如果某个G 阻塞，系统线程 M 会切换执行另外一个G ，所以goroutine阻塞并不会导致系统线程阻塞，这就大大提高了程序并发处理能力
    + `P`: Processor（Golang实现的调度处理器，每个逻辑CPU只能运行一个 P，可以通过设置 GOMAXPROCS 来控制 P 的个数）
    + `M`: OS Thread (系统线程，用来运行goroutine，同一时刻一个 P 只能运行一个 M，M可以随时创建销毁)
    + `G`: Goroutine（用户级协程，每个 M 支持数十万个G (goroutine）)
2. 每个P 有一个local run queue，同时有一个global run queue，都用来存储可运行的goroutine，goroutine运行结束之后会从queue中剔除。
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/go-goroutine.png?raw=true)
3. 每一轮调度，调度器找到一个可运行的 G 会按照以下代码逻辑。如果某个 P local run queue为空，那它会随机从其它 P 中获取一半可运行goroutine。
   ```
   // runtime->proc.go
   func schedule() {
       // only 1/61 of the time, check the global runnable queue for a G.
       // if not found, check the local queue.
       // if not found,
       //     try to steal from other Ps.
       //     if not, check the global runnable queue.
       //     if not found, poll network.
   }
   ```
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/go-goroutine-steal.png?raw=true)


`总的来说: Go 调度器做了很多工作，避免过多的抢占操作系统的线程`

## ④ 



