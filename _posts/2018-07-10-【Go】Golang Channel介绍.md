---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - Go
---

# 一. Channel简介
`Channel` 是Golang的2大核心之一，类似Linux的管道，为并发Goroutine提供一种同步通信机制，借助于Channel不同的Goroutine之间可以相互通信。

1. 创建channel: `make(chan type)` type表示具体数据类型，除了支持常规的int、float64、string等类型外，还支持struct、interface等
   ```
   ch1 := make(chan int)
   ch2 := make(chan float64)
   ch3 := make(chan string)
   ch4 := make(chan struct{})
   ch5 := make(chan interface{})
   ```
2. 发送数据: `ch <- v`，`ch <-` ch在左边表示发送数据到channel ch中
3. 接收数据: `x := <- ch`，`<- ch` ch在右边表示从channel ch中接收数据

根据发送和接收数据，可以分成以下3类Channel
1. 可以发送&可以接收: make(chan type)，默认创建channel都是可以发送和接收数据
2. 只可以发送: make(chan<- type)，只可以发送数据channel
3. 只可以接收: make(<-chan type)，只可以接收数据channel

和slice类似，channel在make的时候也可以设置队列`容量`，根据队列容量大小，可以分成以下2类Channel
1. 无缓冲channel: make没有设置队列容量或者队列容量为0，表示channel是一个无缓存channel，例如make(chan type)
2. 缓冲channel: make设置队列容量不为0，表示channel是一个缓存channel，例如make(chan type, size)

channel可以被关闭，可以使用内置的`close`函数关闭一个channel。

channel具有以下几个特点
1. `从一个nil channel接收数据会一直阻塞`
   ```
   var c1 chan int
   go func() {
       for {
           v := <- c1
	   fmt.Println(v)  //一直阻塞直到程序退出
       }
   }()

   time.Sleep(5 * time.Second)
   fmt.Println("exit ~")
   ```
2. `如果channel没有数据，只要channel未关闭，从channel中接收数据会一直阻塞直到有数据为止`
   ```
   // 无缓冲channel
   c1 := make(chan int)
   go func() {
        <- c1  //阻塞直到c1有数据
        fmt.Println("receiver from c1")
   }()

   fmt.Println(time.Now().Unix())
   time.Sleep(2* time.Second)
   fmt.Println(time.Now().Unix())
   c1 <- 1
   time.Sleep(5* time.Second)
   fmt.Println("exit ~")
   
   // 有缓冲channel
   c2 := make(chan int, 100)
   go func() {
        <- c2  //阻塞直到c2有数据
        fmt.Println("receiver from c2")
   }()

   fmt.Println(time.Now().Unix())
   time.Sleep(2* time.Second)
   fmt.Println(time.Now().Unix())
   c2 <- 1
   time.Sleep(5* time.Second)
   fmt.Println("exit ~")
   ```
3. `无缓冲channel，发送数据和接收数据同时发生。如果没有receiver接收数据(<- chan)，则sender发送数据会一直阻塞；如果没有sender发送数据(chan <-)，则receiver接收数据会一直阻塞`
   ```
   // 没有receiver接收数据
   c1 := make(chan int)
   go func() {
       c1 <- 1  //阻塞直到有receiver
       fmt.Println("sender 1 to c1")
   }()
   time.Sleep(20 * time.Second)
   fmt.Println("exit ~")
   
   // 没有sender发送数据
   c2 := make(chan int)
   go func() {
       <- c2   //阻塞直到有sender
       fmt.Println("receiver from c2")
   }()
   time.Sleep(20 * time.Second)
   fmt.Println("exit ~")
   ```
4. `有缓冲channel，当队列容器未满的时候sender不会阻塞，当队列容量满的时候sender会阻塞，当队列容量为空的时候receiver会阻塞`
   ```
   c1 := make(chan int, 10)
   go func() {
       for i := 0; i < 100; i++ {
           c1 <- i
	   fmt.Printf("send %d to c1\n", i)   //i为10的时候开始阻塞，直到receiver开始消费
       }
   }()

   time.Sleep(5 * time.Second)
   fmt.Println("sleep 5 seconds done ~")
   go func() {
       for {
           <- c1   //消费完所有数据后，receiver开始阻塞
       }
   }()

   time.Sleep(20 * time.Second)
   fmt.Println("exit ~")
   ```
5. `关闭nil channel会panic (panic: close of nil channel)`
   ```
   var c1 chan int
   close(c1)  //panic: close of nil channel
   fmt.Println("exit ~")
   ```
6. `重复关闭channel会panic (panic: close of closed channel)`
   ```
   c1 := make(chan int)
   close(c1)
   time.Sleep(5 * time.Second)
   close(c1)  //panic: close of closed channel
   fmt.Println("exit ~")
   ```
7. `向已经关闭channel发送数据会panic (panic: send on closed channel)`
   ```
   c1 := make(chan int)
   close(c1)
   time.Sleep(5 * time.Second)
   c1 <- 1   //panic: send on closed channel
   fmt.Println("exit ~")
   ```
8. `从已经关闭的channel读取数据不会阻塞，如果channel为空读到对应类型默认值，例如int默认值是0，指针默认值为nil`
   ```
   c1 := make(chan int, 5)
   go func() {
       for i := 1; i <= 5; i++ {
	   c1 <- i
       }
   }()
   time.Sleep(5 * time.Second)

   close(c1)

   go func() {
       for {
           v := <- c1  
	   fmt.Println(v)  //打印出 1 2 3 4 5 0 0 0 0 ...
       }
   }()

   time.Sleep(5 * time.Second)
   fmt.Println("exit ~")
   ```
9. `channel是一个FIFO队列，发送和接收的顺序是一致的`
   ```
   c1 := make(chan int, 5)
   go func() {
       for i := 1; i <= 5; i++ {
           c1 <- i
       }
   }()

   go func() {
       for {
           v := <- c1
	   fmt.Println(v)  //打印出 1 2 3 4 5
       }
   }()

   time.Sleep(5 * time.Second)
   fmt.Println("exit ~")
   ```
10. `可以使用val, ok := <-ch 方式来判断channel是否关闭，ok为true表示channel未关闭，ok为false表示channel关闭`
    ```
    ch := make(chan int)
    go func() {
       for {
           v, ok := <- ch
	   if !ok {
	       fmt.Println("break ~")
	   }
       }
    }()
    time.Sleep(5 * time.Second)
    close(ch)
    time.Sleep(5 * time.Second)
    fmt.Println("exit ~")
    ```
11. `支持多个goroutine同时消费一个Channel，可用于并发处理场景`
    ```
    ch := make(chan int, 100)
    go func() {
	for {
	    ch <- rand.Intn(100000)
	    time.Sleep(200 * time.Millisecond)
	}
    }()

    for i := 0; i < 10; i++{
	go func(idx int) {
	    for v := range ch {
	        fmt.Println(idx, " goroutine receive data: ", v)
	    }
	}(i)
    }
    time.Sleep(1000 * time.Second)
    ```

# 二. Channel用法
Channel最常见用法主要是以下6种

1. goroutine之间同步通信
2. range迭代
3. select操作
4. timeout超时处理
5. timer计时器
6. ticker定时器

## ① goroutine间同步
Channel用的最多的场景是goroutine之间同步通信

```
func main()  {
    ch := make(chan int)
    go produce(ch)
    go consume(ch)

    time.Sleep(10 * time.Second)
    fmt.Println("exit ~")
}

func produce(ch chan int) {
    for {
	v := rand.Intn(1024)
	if v % 16 == 0 {
   	    ch <- v
	}
    }
}

func consume(ch chan int) {
    for v := range ch {
	fmt.Println(v)
    }
}
```

## ② range迭代
range Channel可以直接取到Channel中的值，当Channel关闭后内部数据读完之后循环自动结束，如果Channel没有被关闭range会一直等待Channel。

```
func consume(ch chan int) {
    for v := range ch {
	fmt.Println(v)
    }
}
```

## ③ select操作
select提供了对多个Channel的统一管理，select 语句可以从多个可读的Channel中`随机选取`一个执行。select一般配合for循环一起使用，select的break只能跳到select这一层，因此使用break的时候一般配置label一起使用。

```
func consume(ch1, ch2, ch3 chan int) {
loop:
    for {
	select {
	case v := <-ch1:
	    fmt.Println("ch1 value: ", v)
	case v := <-ch2:
	    fmt.Println("ch2 value: ", v)
	case <-ch3:
	    fmt.Println("break ~")
	    break loop   //跳到loop这里，相当于跳出for循环
	}
    }
}
```

## ④ timeout超时处理
当我们使用select从Channel中读取数据的时候，支持超时处理

```
func main()  {
    ch := make(chan int)
    go produce(ch)
    go consume(ch)

    time.Sleep(10 * time.Second)
    fmt.Println("exit ~")
}

func produce(ch chan int) {
    for {
	v := rand.Intn(1024)
	if v % 16 == 0 {
	    ch <- v
	}
	time.Sleep(1 * time.Second)
    }
}

func consume(ch chan int) {
    for {
	select {
	case v := <- ch:
	    fmt.Println("ch value: ", v)
	case <- time.After(2 * time.Second):     //如果没有从ch中读取到数据，2秒后会打印超时日志
	    fmt.Println("read from ch timeout")
	}
    }
}
```

## ⑤ timer计时器
timer是标准定时器，在到达时间时仅 `触发一次`

```
done := make(chan struct{})
timer := time.NewTimer(time.Second * 3)

go func() {
    fmt.Printf("Now is %s\n", <-timer.C)    //只会打印一次
    done <- struct{}
}()

<-done
close(done)
```

## ⑥ ticker定时器
timer是循环定时器，到达时间就会`触发`

```
done := make(chan struct{})
tiker := time.NewTicker(time.Second * 3)

go func() {
    for t := range tiker.C {
	fmt.Println(t)          //for循环每3s就会打印一次
	if t.Minute() == 55 {
	    done <- struct{}
	    break
	}
    }
}()

<-done
close(done)
```

# 三. Channel原理
Channel的源码在 Go SDK `src/runtime/chan.go`文件，核心的代码如下

Channel的核心数据结构是`hchan`
```
type hchan struct {
    qcount   uint           // 队列中的元素个数
    dataqsiz uint           // 缓冲队列的固定大小
    buf      unsafe.Pointer // 缓冲数组，用来存储缓存数据
    elemsize uint16 // 元素数据类型大小
    closed   uint32 // 是否关闭
    elemtype *_type // 元素数据类型
    sendx    uint   // 下一次发送的 index
    recvx    uint   // 下一次接收的 index
    recvq    waitq  // 接收者队列
    sendq    waitq  // 发送者队列

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```
      
创建一个Channel `make(chan type)` 调用到的核心代码如下，实际会返回一个 `*hchan`，所以我们可以直接使用Channel变量进行参数传递
```
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // compiler checks this but be safe.
    if elem.size >= 1<<16 {
	throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
	throw("makechan: bad alignment")
    }
    if size < 0 || uintptr(size) > maxSliceCap(elem.size) || uintptr(size)*elem.size > maxAlloc-hchanSize {
	panic(plainError("makechan: size out of range"))
    }

    // Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
    // buf points into the same allocation, elemtype is persistent.
    // SudoG's are referenced from their owning thread so they can't be collected.
    // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
    var c *hchan
    switch {
    case size == 0 || elem.size == 0:
	// Queue or element size is zero.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
	// Race detector uses this location for synchronization.
	c.buf = c.raceaddr()
    case elem.kind&kindNoPointers != 0:
	// Elements do not contain pointers.
	// Allocate hchan and buf in one call.
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
	c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
	// Elements contain pointers.
	c = new(hchan)
	c.buf = mallocgc(uintptr(size)*elem.size, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
    }
    return c
}
```

往Channel发送一个数据 `c <- x` 实际调用的核心代码如下
```
// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
     chansend(c, elem, true, getcallerpc())
}

/*
 * generic single channel send/recv
 * If block is not nil,
 * then the protocol will not
 * sleep but return if it could
 * not complete.
 *
 * sleep can wake up with g.param == nil
 * when a channel involved in the sleep has
 * been closed.  it is easiest to loop and re-run
 * the operation; we'll see that it's now closed.
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 往nil channel发数据，gopark 会将当前 goroutine 休眠
    if c == nil {
	if !block {
	    return false
	}
	gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
	throw("unreachable")
    }
	
    if debugChan {
	print("chansend: chan=", c, "\n")
    }

    if raceenabled {
	racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
    }

    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not closed, we observe that the channel is
    // not ready for sending. Each of these observations is a single word-sized read
    // (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
    // Because a closed channel cannot transition from 'ready for sending' to
    // 'not ready for sending', even if the channel is closed between the two observations,
    // they imply a moment between the two when the channel was both not yet closed
    // and not ready for sending. We behave as if we observed the channel at that moment,
    // and report that the send cannot proceed.
    //
    // It is okay if the reads are reordered here: if we observe that the channel is not
    // ready for sending and then observe that it is not closed, that implies that the
    // channel wasn't closed during the first observation.
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
	(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
	return false
    }

    var t0 int64
    if blockprofilerate > 0 {
	t0 = cputicks()
    }

    lock(&c.lock)

    // 往已经关闭的channel发送数据，直接panic
    if c.closed != 0 {
	unlock(&c.lock)
	panic(plainError("send on closed channel"))
    }
    
    // 有 goroutine 阻塞在 channel 上，直接将数据发送给该 goroutine。
    if sg := c.recvq.dequeue(); sg != nil {
  	// Found a waiting receiver. We pass the value we want to send
	// directly to the receiver, bypassing the channel buffer (if any).
	send(c, sg, ep, func() { unlock(&c.lock) }, 3)
	return true
    }

    // 当前 hchan.buf 还有可用空间，将数据放到 buffer 里面
    if c.qcount < c.dataqsiz {
	// Space is available in the channel buffer. Enqueue the element to send.
	qp := chanbuf(c, c.sendx)
	if raceenabled {
	    raceacquire(qp)
	    racerelease(qp)
	}
	typedmemmove(c.elemtype, qp, ep)
	c.sendx++
	if c.sendx == c.dataqsiz {
	    c.sendx = 0
	}
	c.qcount++
	unlock(&c.lock)
	return true
    }

    if !block {
	unlock(&c.lock)
	return false
    } 
    
    // 当前 hchan.buf 已满，阻塞当前 goroutine
    // Block on the channel. Some receiver will complete our operation for us.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
	mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg)
    goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
    
    // someone woke us up.
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if gp.param == nil {
    	if c.closed == 0 {
	    throw("chansend: spurious wakeup")
	}
	panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
	blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```

往Channel接收一个数据 `<- c` 实际调用的核心代码如下
```
// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}

// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: don't need to check ep, as it is always on the stack
    // or is new memory allocated by reflect.

    if debugChan {
	print("chanrecv: chan=", c, "\n")
    }

    // 从一个nil channel接收数据，gopark 会将当前 goroutine 休眠
    if c == nil {
	if !block {
	    return
	}
	gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
	throw("unreachable")
    }

    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not ready for receiving, we observe that the
    // channel is not closed. Each of these observations is a single word-sized read
    // (first c.sendq.first or c.qcount, and second c.closed).
    // Because a channel cannot be reopened, the later observation of the channel
    // being not closed implies that it was also not closed at the moment of the
    // first observation. We behave as if we observed the channel at that moment
    // and report that the receive cannot proceed.
    //
    // The order of operations is important here: reversing the operations can lead to
    // incorrect behavior when racing with a close.
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
	c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
	atomic.Load(&c.closed) == 0 {
	return
    }

    var t0 int64
    if blockprofilerate > 0 {
	t0 = cputicks()
    }

    lock(&c.lock)

    // 如果channel已经关闭，同时没有其它数据返回默认值，使用 ok-idiom 方式读取的时候，第二个参数返回 false
    if c.closed != 0 && c.qcount == 0 {
	if raceenabled {
	    raceacquire(c.raceaddr())
	}
	unlock(&c.lock)
	if ep != nil {
	    typedmemclr(c.elemtype, ep)
	}
	return true, false
    }

    // 当前有发送 goroutine 阻塞在 channel 上，buf 已满
    if sg := c.sendq.dequeue(); sg != nil {
	// Found a waiting sender. If buffer is size 0, receive value
	// directly from sender. Otherwise, receive from head of queue
	// and add sender's value to the tail of the queue (both map to
	// the same buffer slot because the queue is full).
	recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
	return true, true
    }

    // buf 中有可用数据
    if c.qcount > 0 {
	// Receive directly from queue
	qp := chanbuf(c, c.recvx)
	if raceenabled {
	    raceacquire(qp)
	    racerelease(qp)
	}
	if ep != nil {
	    typedmemmove(c.elemtype, ep, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
	    c.recvx = 0
	}
	c.qcount--
	unlock(&c.lock)
	return true, true
    }

    if !block {
	unlock(&c.lock)
	return false, false
    }

    // buf 为空，阻塞
    // no sender available: block on this channel.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
	mysg.releasetime = -1
    }
	
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg)
    goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

    // someone woke us up
    if mysg != gp.waiting {
	throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
	blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, !closed
}
```

关闭Channel `close(chan)` 调用的核心代码如下
```
func closechan(c *hchan) {
    // 关闭一个nil channel，直接panic
    if c == nil {
	panic(plainError("close of nil channel"))
    }

    lock(&c.lock)
    
    // 重复关闭 channel，直接panic
    if c.closed != 0 {
	unlock(&c.lock)
	panic(plainError("close of closed channel"))
    }

    if raceenabled {
	callerpc := getcallerpc()
	racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
	racerelease(c.raceaddr())
    }

    c.closed = 1
    var glist *g

    // 唤醒 recvq 队列里面的阻塞 goroutine
    // release all readers
    for {
	sg := c.recvq.dequeue()
	if sg == nil {
	    break
	}
	
	if sg.elem != nil {
	    typedmemclr(c.elemtype, sg.elem)
	    sg.elem = nil
	}
	
	if sg.releasetime != 0 {
	    sg.releasetime = cputicks()
	}
	
	gp := sg.g
	gp.param = nil
	if raceenabled {
	    raceacquireg(gp, c.raceaddr())
	}
	gp.schedlink.set(glist)
	glist = gp
    }

    // 唤醒 sendq 队列里面的阻塞 goroutine
    // release all writers (they will panic)
    for {
	sg := c.sendq.dequeue()
	if sg == nil {
	    break
	}
	sg.elem = nil
	if sg.releasetime != 0 {
	    sg.releasetime = cputicks()
	}
	gp := sg.g
	gp.param = nil
	if raceenabled {
	    raceacquireg(gp, c.raceaddr())
	}
	gp.schedlink.set(glist)
	glist = gp
    }

    unlock(&c.lock)

    // 所有的 goroutine 放到 glist 队列中，最后唤醒 glist 队列中的 goroutine。
    // Ready all Gs now that we've dropped the channel lock.
    for glist != nil {
	gp := glist
	glist = glist.schedlink.ptr()
	gp.schedlink = 0
	goready(gp, 3)
    }
}
```
