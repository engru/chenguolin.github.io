---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - LevelDB
---

# 一. leveldb简介
  leveldb是google两位工程师实现的单机版k-v存储系统，具有以下几个特点

1. key和value都是任意的字节数组，支持内存和持久化存储
2. 数据都是按照key排序
3. 用户可以重写排序函数
4. 包含基本的数据操作接口，Put(key,value)，Get(key)，Delete(key)
5. 多操作可以当成一次原子操作
6. 用户可以通过生成snapshot，使得读取操作不受写操作影响，读取过程中看到最终数据一致性
7. 支持迭代器对数据的操作
8. 数据使用snappy自动压缩
9. 外部操作（如文件系统操作等）通过一个虚拟接口使用，用户可以对操作系统进行定制相应操作

# 二. leveldb局限性
1. leveldb非关系型数据库，不支持SQL查询也不支持索引
2. 同一时间只支持单进程(支持多线程)访问db
3. 不支持客户端-服务器模型，用户需要自己封装

# 三. leveldb基本框架
  levelDb本质上是一套存储系统以及在这套存储系统上提供的一些操作接口。为了便于理解整个系统及其处理流程，我们可以从两个不同的角度来看待LevleDb：静态角度和动态角度。从静态角度，可以假想整个系统正在运行过程中（不断插入删除读取数据），此时我们给LevelDb照相，从照片可以看到之前系统的数据在内存和磁盘中是如何分布的，处于什么状态等；从动态的角度，主要是了解系统是如何写入一条记录，读出一条记录，删除一条记录的，同时也包括除了这些接口操作外的内部操作比如compaction，系统运行时崩溃后如何恢复系统等等方面
  leveldb做为存储系统，在整个系统运行过程中，基本的框架如下所示
  ![](https://img-blog.csdn.net/20160130211357097?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
  
  如图所示，leveldb的存储介质分为内存和磁盘两种。内存中有memtable和immutable memtable；磁盘中有log文件,manifest文件，Current文件以及分level的sstable文件；

1. 当用户往db插入一条key-value数据的时候，会先写log文件，当写log成功之后再把当前记录写到memtable中。为什么写数据的时候要先写log文件呢，主要是因为新插入的数据会保存在内存中，为了防止系统崩溃导致新插入数据丢失，因此要先写log文件保证落地之后，再写内存。这样即使系统崩溃了，也能够从log中恢复出来。
2. memtable中的数据是可读可写，当memtable的数据量达到一个数据量之后。当前的memtable变成了immutable memtable，只读不可修改。重新生成新的memtable和log文件，新来的数据写到新的log和memtable中。
3. immutable memtable中的数据会被dump到磁盘中的sstable文件，磁盘中的sstable文件是有层级的，第一层level0到第n层leveln...，每个level都有很多sstable文件，每个文件都是按照key排好序。注意了level0和其它level不一样，level0中的sstable文件的key有可能重复，其它level的sstable文件的key保证不会有重复。
4. 由于每个level中有许多sstable文件，每个sstable文件都有key range。所以需要一个文件来保存当前所有level中sstabel的key range。manifest文件主要就是用来存储每个level中的sstable的信息
5. 随着系统不断的运行，每个level中的sstable文件可能会越来越多，这个时候db会自动把同一个level或者不同level中的sstable文件会进行merge。这个时候manifest就会发生变化，因此我们需要一个Current文件来记录当前最新的manifest文件。

