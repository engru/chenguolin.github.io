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
1. `内存消耗`: 创建一个goroutine比创建一个线程需要的内存更少，创建一个线程需要`1MB`的内存空间，而创建一个goroutine只需要`2KB`的内存空间，比线程节省了500倍。在Golang程序中一个线程可以承载上千个goroutine。
2. `启动和销毁的成本`: 线程的启动和销毁必须要和操作系统进行交互，goroutine的启动和销毁都是通过Go运行时，Go运行时负责管理调度、垃圾回收、以及goroutine运行时环境配置，goroutine相对于线程来说成本低很多
   ![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/goroutine-vs-thread.png?raw=true)

## ③ Goroutine调度本质

## ④ 


