---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Go
---

# 一. Goroutine
**Goroutine**是Golang2个核心的设计之一，Goroutine在Golang里面指的是**协程**。我们知道线程属于系统层面，通常来说创建一个新的线程会消耗较多的资源且管理不易。而Goroutine就像轻量级的线程，但我们称其为协程，一个Go程序可以运行超过数万个Goroutine，并且这些性能都是原生级的，随时都能够关闭、结束。

在内置的官方包中也不时能够看见Goroutine的应用，像是net/http中用来监听网络服务的函数实际上是创建一个不断运行循环的Goroutine。

我们可以通过**runtime.GOMAXPROCS**来设置同时执行的最大CPU数，GO默认是使用一个CPU核的，除非设置runtime.GOMAXPROCS  
`那么在多核环境下什么情况下设置runtime.GOMAXPROCS会比较好的提高速度呢？适合于CPU密集型、并行度比较高的情景。如果是IO密集型，CPU之间的切换也会带来性能的损失。`
```
func main() {
    num := runtime.NumCPU()    # 本地机器的逻辑CPU个数
    runtime.GOMAXPROCS(num)    # 设置可同时执行的最大CPU数
    fmt.Println(num)
}
```

# 二. 并发
Goroutine最大的一个功能就是实现高并发，通过Goroutine能够让程序以异步的方式运行，而不需要担心一个函数导致程序中断，因此Go语言也非常地适合网络服务。

我们通过go让其中一个函数异步运行，如此就不需要等待该函数运行完后才能运行下一个函数。
```
func main() {
    # 通过 `go` 我们可以把这个函数异步执行，这样就不会阻塞往下执行。
    go loop()
    # 执行 Other
}
```

# 三. Recover
虽然Goroutine能够实现高并发，但是如果某个Goroutine panic了，而且这个Goroutine里面没有捕获recover，那么整个进程就会挂掉。所以，好的习惯是每当go产生一个goroutine，就需要写下recover。
```
func domainPut(num int) {
    defer func() {
        err := recover()
        if err != nil {
            fmt.Println("panic error.")
        }
    }()
    panic("error....")
}

func main() {
    for i := 0; i < 10; i++ {
        domainName := i
        go domainPut(domainName)
    }
    time.Sleep(time.Second * 2)
}
```

# 四. 规范
使用Goroutine实现高并发有一些规范开发必须要注意，否则很容易带来困扰

1. `Golang主程序必须要等待所有的Goroutine结束才能够退出，否则如果先退出主程序会导致所有的Goroutine可能未执行结束就退出了`
   ```
   package main

   import (
       "sync"
       "fmt"
       "time"
   )

   func calc(w *sync.WaitGroup, i int)  {
       defer func() {
           err := recover()
           if err != nil {
             fmt.Println("panic error.")
           }
       }()
    
       fmt.Println("calc: ", i)
       time.Sleep(time.Second)
       w.Done()
   }

   func main()  {
       # WaitGroup能够一直等到所有的goroutine执行完成，并且阻塞主线程的执行，直到所有的goroutine执行完成。
       wg := sync.WaitGroup{}    
       for i:=0; i<10; i++ {
           wg.Add(1)
           go calc(&wg, i)
       }
       # 阻塞主线程等到所有的goroutine执行完成
       wg.Wait()
       fmt.Println("all goroutine finish")
   }
   ```

2. `每个Goroutine都要有recover机制，因为当一个Goroutine抛panic的时候只有自身能够捕捉到其它Goroutine是没有办法捕捉的。如果没有recover机制，整个进程会crash`
3. `recover只能在defer里面生效，如果不是在defer里调用，会直接返回nil`
4. `Goroutine发生panic时，只会调用自身的defer，所以即便主Goroutine里写了recover逻辑，也无法recover`
   ```
   # Goroutine里面需要写defer recover机制
   go func() {
      defer func() {
         err := recover()
         if err != nil {
            fmt.Println("panic error.")
         }
      }()
      # other code
   }()
   ```

