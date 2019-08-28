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
2. `启动和销毁的成本`: 线程的启动和销毁必须要和操作系统内核进行交互，goroutine的启动和销毁都是通过Go runtime，Go runtime负责管理调度、垃圾回收、goroutine创建等，goroutine相对于线程来说启动和销毁成本低很多
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-vs-thread.png?raw=true)
3. `上下文切换成本`: goroutine和线程最大的区别是上下文切换成本，线程是抢占式调度的，当线程运行时间片到了之后就会被另外一个可运行线程抢占，这个过程需要涉及到上下文切换，线程需要保存所有的寄存器，包括通用寄存器、程序计数器等。goroutine调度是协作式的，goroutine的调度不需要和操作系统内核进行交互，当goroutine进行切换的时候只需要保存少量的程序计数器，成本比线程低很多。

## ③ Goroutine调度本质

## ④ 


