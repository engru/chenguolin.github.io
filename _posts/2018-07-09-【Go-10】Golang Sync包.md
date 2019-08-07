---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Go
---

# 一. 前言
Golang sync包提供了基础的异步操作方法，包括`互斥锁Mutex`，`执行一次Once`和`并发等待组WaitGroup`。  
本文主要介绍sync包提供的这些功能的基本使用方法。

1. Mutex: 互斥锁
2. RWMutex：读写锁
3. WaitGroup：并发等待组
4. Once：执行一次
5. Cond：信号量
6. Pool：临时对象池
7. Map：自带锁的map

# 二. sync.Mutex
`sync.Mutex`称为`互斥锁`，常用在并发编程里面。互斥锁需要保证的是同一个时间段内不能有多个并发协程同时访问某一个资源(临界区)。  
`sync.Mutex`有2个函数`Lock`和`UnLock`分别表示获得锁和释放锁。

```
func (m *Mutex) Lock()
func (m *Mutex) UnLock()
```

`sync.Mutex初始值为UnLock状态，并且sync.Mutex常做为其它结构体的匿名变量使用。`

举个例子: 我们经常使用网上支付购物东西，就会出现同一个银行账户在某一个时间既有支出也有收入，那银行就得保证我们余额准确，保证数据无误。  
我们可以简单的实现银行的支出和收入来说明Mutex的使用

```
type Bank struct {
	sync.Mutex
	balance map[string]float64
}

// In 收入
func (b *Bank) In(account string, value float64) {
        // 加锁 保证同一时间只有一个协程能访问这段代码
	b.Lock()
	defer b.Unlock()

	v, ok := b.balance[account]
	if !ok {
		b.balance[account] = 0.0
	}

	b.balance[account] += v
}

// Out 支出
func (b *Bank) Out(account string, value float64) error {
        // 加锁 保证同一时间只有一个协程能访问这段代码
	b.Lock()
	defer b.Unlock()

	v, ok := b.balance[account]
	if !ok || v < value {
		return errors.New("account not enough balance")
	}

	b.balance[account] -= value
	return nil
}
```

# 三. sync.RWMutex
`sync.RWMutex`称为读写锁是`sync.Mutex`的一种变种，RWMutex来自于计算机操作系统非常有名的`读者写者问题`。  
`sync.RWMutex目的是为了能够支持多个并发协程同时读取某一个资源，但只有一个并发协程能够更新资源。也就是说读和写是互斥的，写和写也是互斥的，读和读是不互斥的。`

总结起来如下  
1. 当有一个协程在读的时候，所有写的协程必须等到所有读的协程结束才可以获得锁进行写操作。
1. 当有一个协程在读的时候，所有读的协程不受影响都可以进行读操作。
2. 当有一个协程在写的时候，所有读、写的协程必须等到写的协程结束才可以获得锁进行读、写操作。

`RWMutex`有5个函数，分别为读和写提供锁操作
```
写操作
func (rw *RWMutex) Lock()
func (rw *RWMutex) Unlock()

读操作
func (rw *RWMutex) RLock()
func (rw *RWMutex) RUnlock()

RLocker()能获取读锁，然后传递给其他协程使用。
func (rw *RWMutex) RLocker() Locker
```

举个例子，`sync.Mutex`一节例子里面我们没有提供查询操作，如果用Mutex互斥锁就没有办法支持多人同时查询，所以我们使用`sync.RWMutex`来改写这个代码
```
type Bank struct {
	sync.RWMutex
	balance map[string]float64
}

func (b *Bank) In(account string, value float64) {
	b.Lock()
	defer b.Unlock()

	v, ok := b.balance[account]
	if !ok {
		b.balance[account] = 0.0
	}

	b.balance[account] += v
}

func (b *Bank) Out(account string, value float64) error {
	b.Lock()
	defer b.Unlock()

	v, ok := b.balance[account]
	if !ok || v < value {
		return errors.New("account not enough balance")
	}

	b.balance[account] -= value
	return nil
}

func (b *Bank) Query(account string) float64 {
	b.RLock()
	defer b.RUnlock()

	v, ok := b.balance[account]
	if !ok {
		return 0.0
	}

	return v
}
```

# 四. sync.WaitGroup
`sync.WaitGroup`指的是等待组，在Golang并发编程里面非常常见，指的是`等待一组工作完成后，再进行下一组工作`。

`sync.WaitGroup`有3个函数
```
func (wg *WaitGroup) Add(delta int)  Add添加n个并发协程
func (wg *WaitGroup) Done()  Done完成一个并发协程
func (wg *WaitGroup) Wait()  Wait等待其它并发协程结束
```

`sync.WaitGroup`在Golang编程里面最常用于协程池，下面这个例子会同时启动1000个并发协程。
```
func main() {
     wg := &sync.WaitGroup{}
     for i := 0; i < 1000; i++ {
         wg.Add(1)
         go func() {
	     defer func() {
		wg.Done()
	     }()
	     time.Sleep(1 * time.Second)
	     fmt.Println("hello world ~")
	 }()
     }
     // 等待所有协程结束
     wg.Wait()
     fmt.Println("WaitGroup all process done ~")
}
```

`sync.WaitGroup`没有办法指定最大并发协程数，在一些场景下会有问题。例如操作数据库场景下，我们不希望某一些时刻出现大量连接数据库导致数据库不可访问。所以，为了能够控制最大的并发数，推荐使用`github.com/remeh/sizedwaitgroup`，用法和`sync.WaitGroup`非常类似。

下面这个例子最多只有10个并发协程，如果已经达到10个并发协程，只有某一个协程执行了Done才能启动一个新的协程。
```
import  "github.com/remeh/sizedwaitgroup"

func main() {
     # 最大10个并发
     wg := sizedwaitgroup.New(10)
     for i = 0; i < 1000; i++ {
         wg.Add()
         go func() {
	     defer func() {
		wg.Done()
	     }()
	     time.Sleep(1 * time.Second)
	     fmt.Println("hello world ~")
	 }()
     }
     // 等待所有协程结束
     wg.Wait()
     fmt.Println("WaitGroup all process done ~")
}
```

# 五. sync.Once
`sync.Once`指的是只执行一次的对象实现，常用来控制某些函数只能被调用一次。`sync.Once`的使用场景例如单例模式、系统初始化。  
例如并发情况下多次调用channel的close会导致panic，解决这个问题我们可以使用sync.Once来保证close只会被执行一次。

`sync.Once`的结构如下所示，只有一个函数。使用变量`done`来记录函数的执行状态，使用`sync.Mutex`和`sync.atomic`来保证线程安全的读取`done`。
```
type Once struct {
	m    Mutex     #互斥锁
	done uint32    #执行状态
}

func (o *Once) Do(f func())
```

举个例子，1000个并发协程情况下只有一个协程会执行到fmt.Printf，多次执行的情况下输出的内容还不一样，因为这取决于哪个协程先调用到该匿名函数。
```
func main() {
     once := &sync.Once{}

     for i := 0; i < 1000; i++ {
	 go func(idx int) {
	    once.Do(func() {
		time.Sleep(1 * time.Second)
		fmt.Printf("hello world index: %d", idx)
	    })
	 }(i)
     }

     time.Sleep(5 * time.Second)
}
```

# 六. sync.Cond
`sync.Cond`指的是同步条件变量，一般需要与互斥锁组合使用，本质上是一些正在等待某个条件的协程的同步机制。

```
// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
    return &Cond{L: l}
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}
```

`sync.Cond`有3个函数`Wait`、`Signal`、`Broadcast`
```
// Wait 等待通知
func (c *Cond) Wait()
// Signal 单播通知
func (c *Cond) Signal()
// Broadcast 广播通知
func (c *Cond) Broadcast()
```

举个例子，`sync.Cond`用于并发协程条件变量。
```
var sharedRsc = make(map[string]interface{})
func main() {
    var wg sync.WaitGroup
    wg.Add(2)
    m := sync.Mutex{}
    c := sync.NewCond(&m)
    
    go func() {
        // this go routine wait for changes to the sharedRsc
        c.L.Lock()
        for len(sharedRsc) == 0 {
            c.Wait()
        }
        fmt.Println(sharedRsc["rsc1"])
        c.L.Unlock()
        wg.Done()
    }()

    go func() {
        // this go routine wait for changes to the sharedRsc
        c.L.Lock()
        for len(sharedRsc) == 0 {
            c.Wait()
        }
        fmt.Println(sharedRsc["rsc2"])
        c.L.Unlock()
        wg.Done()
    }()

    // this one writes changes to sharedRsc
    c.L.Lock()
    sharedRsc["rsc1"] = "foo"
    sharedRsc["rsc2"] = "bar"
    c.Broadcast()
    c.L.Unlock()
    wg.Wait()
}
```

# 七. sync.Pool
`sync.Pool` 指的是临时对象池，Golang和Java具有GC机制，因此很多开发者基本上都不会考虑内存回收问题，不像C++很多时候开发需要自己回收对象。  
Gc是一把双刃剑，带来了编程的方便但同时也增加了运行时开销，使用不当可能会严重影响程序的性能，因此性能要求高的场景不能任意产生太多的垃圾。  
`sync.Pool` 正是用来解决这类问题的，Pool可以作为临时对象池来使用，不再自己单独创建对象，而是从临时对象池中获取出一个对象。  
`sync.Pool` 是并发安全的，允许多个goroutine同时使用 

`sync.Pool`有2个函数`Get`和`Put`，Get负责从临时对象池中取出一个对象，Put用于结束的时候把对象放回临时对象池中。
```
func (p *Pool) Get() interface{}
func (p *Pool) Put(x interface{})
```

看一个官方例子
```
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func timeNow() time.Time {
    return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
    // 获取临时对象，没有的话会自动创建
    b := bufPool.Get().(*bytes.Buffer)
    b.Reset()
    b.WriteString(timeNow().UTC().Format(time.RFC3339))
    b.WriteByte(' ')
    b.WriteString(key)
    b.WriteByte('=')
    b.WriteString(val)
    w.Write(b.Bytes())
    // 将临时对象放回到 Pool 中
    bufPool.Put(b)
}

func main() {
    Log(os.Stdout, "path", "/search?q=flowers")
}
```

从上面的例子我们可以看到创建一个Pool对象并不能指定大小，所以sync.Pool的缓存对象数量是没有限制的(只受限于内存)，那sync.Pool是如何控制缓存临时对象数的呢？  
`sync.Pool在init的时候注册了一个poolCleanup函数，它会清除所有的pool里面的所有缓存的对象，该函数注册进去之后会在每次Gc之前都会调用，因此sync.Pool缓存的期限只是两次Gc之间这段时间。正因Gc的时候会清掉缓存对象，所以不用担心pool会无限增大的问题。`

正因为如此`sync.Pool`适合用于缓存临时对象，而不适合用来做持久保存的对象池(连接池等)。

# 八. sync.Map
Go在1.9版本之前自带的map对象是不具有并发安全的，很多时候我们都得自己封装支持并发安全的Map结构，如下所示给map加个读写锁sync.RWMutex。

```
type MapWithLock struct {
    sync.RWMutex
    M map[string]Kline
}
```

Go1.9版本新增了`sync.Map`它是原生支持并发安全的map，sync.Map封装了更为复杂的数据结构实现了比之前`加读写锁锁map`更优秀的性能。

`sync.Map`总共5个函数，用法和原生的map差点很多
```
// 查询一个key
func (m *Map) Load(key interface{}) (value interface{}, ok bool)
// 设置key value
func (m *Map) Store(key, value interface{})
// 如果key存在则返回key对应的value，否则设置key value
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)
// 删除一个key
func (m *Map) Delete(key interface{})
// 遍历map，仍然是无序的
func (m *Map) Range(f func(key, value interface{}) bool)
```



