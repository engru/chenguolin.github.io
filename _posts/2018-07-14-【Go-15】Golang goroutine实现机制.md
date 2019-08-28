---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Go
---

# 一. Goroutine定义
1. Goroutine指的是Go的协程，受`Coroutine`启发把C改为G，因此得名为Goroutine，Goroutine是Golang用来实现并发编程机制。
2. Goroutine是轻量级线程，和系统线程类似，只不过它是在语言级别上实现的。系统线程上下文切换的核心是逻辑CPU，而Goroutine上下文切换的核心是系统线程 M。
3. 在Golang程序中我们只需要使用 `go` 就可以创建一个Goroutine。
   ```
   func learning() {  
       fmt.Println("My first goroutine")
   }
   
   func main() {  
       go learning()
       /* we are using time sleep so that the main program does not terminate before the execution of goroutine.*/
       time.Sleep(1 * time.Second)
       fmt.Println("main function")
   }
   ```

# 二. Goroutine和线程的区别
1. `内存消耗`: 创建一个Goroutine比创建一个线程需要的内存更少，创建一个线程需要`1MB`的内存空间，而创建一个Goroutine只需要`2KB`的内存空间，比线程节省了500倍，所以我们可以在Golang程序中创建数百万个Goroutine。
2. `启动和销毁的成本`: 线程的启动和销毁必须要和操作系统内核进行交互，Goroutine的启动和销毁都是通过Go runtime，Go runtime负责管理调度、垃圾回收、Goroutine创建等，Goroutine相对于线程来说启动和销毁成本低很多。
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-vs-thread.png?raw=true)
3. `上下文切换成本`: Goroutine和线程最大的区别是上下文切换成本，线程是抢占式调度的，当线程运行CPU时间片到了之后就会被另外一个可运行线程抢占，这个过程需要涉及到上下文切换，线程需要保存所有的寄存器，包括通用寄存器、程序计数器等。Goroutine调度是协作式的，Goroutine的调度不需要和操作系统内核进行交互，当Goroutine进行切换的时候只需要保存少量的程序计数器，成本比线程低很多。
4. `状态`: Goroutine和系统线程一样同样有3种状态 `Waiting`、`Runnable`、`Executing`

# 三. Go调度器
1. Go使用以下3个实体描述Goroutine调度，通过这3个Go就可以实现`x:y`的调度，也就是 x 个Goroutine可以运行在 y 个系统线程上，任何时候每个系统线程 M 都可以执行一个G，如果某个G 阻塞，系统线程 M 会切换执行另外一个G ，所以Goroutine阻塞并不会导致系统线程阻塞，这就大大提高了程序并发处理能力
    + `P`: Processor（Golang实现的调度处理器，每个逻辑CPU只能运行一个 P，可以通过设置 GOMAXPROCS 来控制 P 的个数）
    + `M`: OS Thread (系统线程，用来运行Goroutine，同一时刻一个 P 只能运行一个 M，M可以随时创建销毁)
    + `G`: Goroutine（用户级协程，每个 M 支持数十万个G (Goroutine）)
2. 每个P 有一个local queue，同时有一个global queue，都用来存储可运行的Goroutine，Goroutine运行结束之后会从queue中剔除。
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/go-goroutine.png?raw=true)
3. 每一轮调度，调度器找到一个可运行的 G 会按照以下代码逻辑。如果某个 P local queue为空，那它会随机从其它 P 中获取一半可运行Goroutine。一旦找到一个可运行的 G，那么它会被执行直到被阻塞。
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
4. Go中即使产生了成千上万个Goroutine，如果大多数Goroutine都因为一些原因阻塞了，这样也不会导致系统资源浪费，因为Go runtime会切换去执行可执行的Goroutine。

`总的来说: Go 调度器做了很多工作，避免操作系统的线程之间有过多的抢占行为，从而提高整体性能`

# 四. Goroutine调度举例
1. Go调度器包括2种类型Queue: GRQ (Global Run Queue) 和 LRQ (Local Run Queue)，每个P都有一个LRQ，每个P同一时刻只能运行一个M，每个M上运行G，G在M上实现调度上下文切换。
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-1.png?raw=true)
2. Goroutine异步系统调用（例如网络调用）  
   G1正在M上运行，准备去执行网络调用
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-2.png?raw=true)
   
   G1切换去执行网络调用，M切换去执行G2
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-3.png?raw=true)
   
   G1执行完成之后push到LRQ，最大的好处是异步网络调用不需要创建新的M
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-4.png?raw=true)
3. Goroutine同步系统调用（例如file I/O）  
   G1正在M上运行，准备去执行阻塞系统调用
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-5.png?raw=true)

   G1和M1阻塞，P上切换新M执行（如果M2已经存在会比创建一个新的更快），M2执行G2
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-6.png?raw=true)

   G1执行完成之后push到LRQ，M1放在一边以备将来使用
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-7.png?raw=true)
4. Goroutine work-stealing  
   有2个P，每个P LRQ有3个G
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-8.png?raw=true)
   
   P1 LRQ为空，P1需要从其它P获取G
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-9.png?raw=true)
   
   P1从P2 LRQ获取一半G，开始执行G3
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-10.png?raw=true)
   
   P2 LRQ为空，计划从P1获取G，发现P1 LRQ也为空，因此从GRQ获取
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-11.png?raw=true)
   
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-12.png?raw=true)

# 五. 并行与并发
1. `并发`: 多个线程在同一个逻辑CPU核上轮流使用CPU时间片，如下图所示G2、G3、G5属于并发
2. `并行`: 多个线程在多个逻辑CPU核上同时运行，如下图所示G1和G2属于并行

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-figure-13.png?raw=true)


