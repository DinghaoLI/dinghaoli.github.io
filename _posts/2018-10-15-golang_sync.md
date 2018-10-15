---
layout: post
title:  "Golang sync包"
date:   2018-10-15 03:00:00 +0800
categories: Golang
tags: Golang

---

本文参考：

[The Go scheduler](http://morsmachine.dk/go-scheduler)   
[Golang 的 goroutine 是如何实现的？](https://www.zhihu.com/question/20862617)   
[浅谈 Golang sync 包的相关使用方法](https://deepzz.com/post/golang-sync-package-usage.html)     
[golang - 条件变量(Cond)](https://cyent.github.io/golang/goroutine/sync_cond/) 

---

Golang的sync包经常用于并发场景，本文介绍sync包的大致用法，许多特性和功能其实与POSIX API中的内容大同小异，只是sync包针对的是goroutine（[什么是goroutine？](https://dinghaoli.github.io/2018/10/goroutine/)），相当于是golang层的用于并行场景的API。

---
## **1.互斥锁 Mutex**

<br/>

```
func (m *Mutex) Lock()
func (m *Mutex) Unlock()

```

互斥锁只能同时被一个 goroutine 锁定，其它 goroutine 将阻塞直到互斥锁被解锁（重新争抢对互斥锁的锁定）

```
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    ch := make(chan struct{}, 2)

    var l sync.Mutex
    go func() {
        l.Lock()
        defer l.Unlock()
        fmt.Println("goroutine1: 我会锁定大概 2s")
        time.Sleep(time.Second * 2)
        fmt.Println("goroutine1: 我解锁了，你们去抢吧")
        ch <- struct{}{}
    }()

    go func() {
        fmt.Println("groutine2: 等待解锁")
        l.Lock()
        defer l.Unlock()
        fmt.Println("goroutine2: 哈哈，我锁定了")
        ch <- struct{}{}
    }()

    // 等待 goroutine 执行结束
    for i := 0; i < 2; i++ {
        <-ch
    }
}

```

---

## **2.读写锁 RWMutex**

<br/>

读写操作的互斥锁，读写锁与互斥锁最大的不同就是可以分别对读、写进行锁定。一般用在大量读操作、少量写操作的情况：

```
func (rw *RWMutex) Lock()
func (rw *RWMutex) Unlock()

func (rw *RWMutex) RLock()
func (rw *RWMutex) RUnlock()

```

- 写锁定（Lock），对写操作进行锁定
- 写解锁（Unlock），对写锁定进行解锁
- 读锁定（RLock），对读操作进行锁定
- 读解锁（RUnlock），对读锁定进行解锁


**在首次使用之后，不要复制该读写锁。不要混用锁定和解锁，如：Lock 和 RUnlock、RLock 和 Unlock。因为对未读锁定的读写锁进行读解锁或对未写锁定的读写锁进行写解锁将会引起运行时错误。**

如何理解读写锁呢？

- 同时只能有一个 goroutine 能够获得写锁定。
- 同时可以有任意多个 gorouinte 获得读锁定。
- 同时只能存在写锁定或读锁定（读和写互斥）。

**其实很好理解，我在写的时候，不允许别人同时写（容易竞争错乱），也不允许别人读（容易读错数据）。我在读的时候，大家可以一起读（分享嘛～），但是不能有人写（我们读的人读错了怎么办）。**

例子如下：

```
package main

import (
    "fmt"
    "math/rand"
    "sync"
)

var count int
var rw sync.RWMutex

func main() {
    ch := make(chan struct{}, 10)
    for i := 0; i < 5; i++ {
        go read(i, ch)
    }
    for i := 0; i < 5; i++ {
        go write(i, ch)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }
}

func read(n int, ch chan struct{}) {
    rw.RLock()
    fmt.Printf("goroutine %d 进入读操作...\n", n)
    v := count
    fmt.Printf("goroutine %d 读取结束，值为：%d\n", n, v)
    rw.RUnlock()
    ch <- struct{}{}
}

func write(n int, ch chan struct{}) {
    rw.Lock()
    fmt.Printf("goroutine %d 进入写操作...\n", n)
    v := rand.Intn(1000)
    count = v
    fmt.Printf("goroutine %d 写入结束，新值为：%d\n", n, v)
    rw.Unlock()
    ch <- struct{}{}
}

```


---

## **3.WaitGroup**

<br/>

WaitGroup 用于等待一组 goroutine 结束，用法很简单。它有三个方法：

```
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()

```

其中Done()是Add(-1)的别名。简单的来说，使用Add()添加计数，Done()减掉一个计数，计数不为0, 阻塞Wait()的运行。

例子：同时开三个协程去请求网页， 等三个请求都完成后才继续 Wait 之后的工作。

```

var wg sync.WaitGroup 
var urls = []string{ 
    "http://www.golang.org/", 
    "http://www.google.com/", 
    "http://dinghaoli.github.io", 
} 
for _, url := range urls { 
    // Increment the WaitGroup counter. 
    wg.Add(1) 
    // Launch a goroutine to fetch the URL. 
    go func(url string) { 
        // Decrement the counter when the goroutine completes. 
        defer wg.Done() 
        // Fetch the URL. 
        http.Get(url) 
    }(url) 
} 
// Wait for all HTTP fetches to complete. 
wg.Wait()

```

---

## **4.Cond条件锁**

<br/>

与POSIX标准的用法差不多

```

cond.L.Lock()
cond.L.Unlock()
cond.Wait()
cond.Signal()
cond.Broadcast()

```

- cond.L.Lock()和cond.L.Unlock()：也可以使用lock.Lock()和lock.Unlock()，完全一样，因为是指针转递

- cond.Wait()：Unlock() -> 阻塞等待通知(即等待Signal()或Broadcast()的通知) -> 收到通知 -> Lock()

- cond.Signal()：通知一个Wait()了的，若没有Wait()，也不会报错。Signal()通知的顺序是根据原来加入通知列表(Wait())的先入先出

- cond.Broadcast(): 通知所有Wait()了的，若没有Wait()，也不会报错

有文章提议用与生产者消费者模式。（但是某些实际情况可能会导致最后一个锁永远没人用）。

![Cond](../../../images/article/golang_sync.png) 

**绿色框就是Wait()的实际动作**

---

## **5.Pool 临时对象池**

<br/>

要说Pool的作用的话，还是先得从栈和碓说起。

- 栈区（stack）— 由编译器自动分配释放，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。  
- 堆区（heap） — 一般由程序员分配释放，若程序员不释放，程序结束时可能由OS回收，golang当中就是垃圾回收（GC）。

```

func F() {
    temp := make([]int, 0, 20)
    ...
}

```

类似于上面代码里面的temp变量，只是内函数内部申请的临时变量，并不会作为返回值返回，它就是被编译器申请到栈里面。申请到栈内存好处：函数返回直接释放，不会引起垃圾回收，对性能没有影响。


```

func F() []int{
    a := make([]int, 0, 20)
    return a
}

```

而上面这段代码，申请的代码一模一样，但是申请后作为返回值返回了，编译器会认为变量之后还会被使用，当函数返回之后并不会将其内存归还，那么它就会被申请到堆上面了。申请到堆上面的内存才会引起垃圾回收。

```

func F() {
    a := make([]int, 0, 20)
    b := make([]int, 0, 20000)
    l := 20
    c := make([]int, 0, l)
}


```

a和b代码一样，就是申请的空间不一样大，但是它们两个的命运是截然相反的。a前面已经介绍过，会申请到栈上面，而b，由于申请的内存较大，编译器会把这种申请内存较大的变量转移到堆上面。即使是临时变量，申请过大也会在堆上面申请。

而c，对我们而言其含义和a是一致的，但是编译器对于这种不定长度的申请方式，也会在堆上面申请，即使申请的长度很短。

**实际项目基本都是通过c := make([]int, 0, l)来申请内存，长度都是不确定的。自然而然这些变量都会申请到堆上面了**。Golang使用的垃圾回收算法是『标记——清除』。简单得说，就是程序要从操作系统申请一块比较大的内存，内存分成小块，通过链表链接。**每次程序申请内存，就从链表上面遍历每一小块，找到符合的就返回其地址，没有合适的就从操作系统再申请。如果申请内存次数较多，而且申请的大小不固定，就会引起内存碎片化的问题。申请的堆内存并没有用完，但是用户申请的内存的时候却没有合适的空间提供。这样会遍历整个链表，还会继续向操作系统申请内存。这就能解释我一开始描述的问题，申请一块内存变成了慢语句**。

**申请内存变成了慢语句，解决方法就是使用Pool临时对象池**

```

package main

import (
    "fmt"
    "sync"
    "time"
)

// 一个[]byte的对象池，每个对象为一个[]byte
var bytePool = sync.Pool{
    New: func() interface{} {
        b := make([]byte, 1024)
        return &b
    },
}

func main() {
    a := time.Now().Unix()
    // 不使用对象池
    for i := 0; i < 1000000000; i++ {
        obj := make([]byte, 1024)
        // 丢弃obj
        _ = obj
    }
    b := time.Now().Unix()
    // 使用对象池
    for i := 0; i < 1000000000; i++ {
        obj := bytePool.Get().(*[]byte)
        // 把obj放回Pool
        bytePool.Put(obj)
    }
    c := time.Now().Unix()
    fmt.Println("without pool ", b-a, "s")
    fmt.Println("with    pool ", c-b, "s")
}

```

Output:

```

without pool  20 s
with    pool  15 s

```

新键 Pool 需要提供一个 New 方法，目的是当获取不到临时对象时自动创建一个（不会主动加入到 Pool 中），Get 和 Put 方法都很好理解。

深入了解过 Go 的同学应该知道，Go 的重要组成结构为 M、P、G。Pool 实际上会为每一个操作它的 goroutine 相关联的 P 都生成一个本地池。如果从本地池 Get 对象的时候，本地池没有，则会从其它的 P 本地池获取。因此，Pool 的一个特点就是：可以把由其中的对象值产生的存储压力进行分摊。

它有着以下特点：

- Pool的目的是缓存已分配但未使用的项目以备后用

- 多协程并发安全

- 缓存在Pool里的item会没有任何通知情况下随时被移除，以缓解GC压力

- 池提供了一种方法来缓解跨多个客户端的分配开销。

- 不是所有场景都适合用Pool，如果释放链表是某个对象的一部分，并由这个对象维护，而这个对象只由一个客户端使用，在这个客户端工作完成后释放链表，那么用Pool实现这个释放链表是不合适的。

>官方对Pool的目的描述：
>
>Pool设计用意是在全局变量里维护的释放链表，尤其是被多个 goroutine 同时访问的全局变量。使用Pool代替自己写的释放链表，可以让程序运行的时候，在恰当的场景下从池里重用某项值。sync.Pool一种合适的方法是，为临时缓冲区创建一个池，多个客户端使用这个缓冲区来共享全局资源。另一方面，如果释放链表是某个对象的一部分，并由这个对象维护，而这个对象只由一个客户端使用，在这个客户端工作完成后释放链表，那么用Pool实现这个释放链表是不合适的。

那么 Pool 都适用于什么场景呢？从它的特点来说，适用与无状态的对象的复用，而不适用与如连接池之类的。在 fmt 包中有一个很好的使用池的例子，它维护一个动态大小的**临时输出缓冲区**。

```

package main

import (
    "bytes"
    "io"
    "os"
    "sync"
    "time"
)

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

打印结果：
2006-01-02T15:04:05Z path=/search?q=flowers

```

**总结：**

- 当每个对象的内存小于一定量的时候，不使用pool的性能秒杀使用pool；当内存处于某个量的时候，不使用pool和使用pool性能相当；当内存大于某个量的时候，使用pool的优势就显现出来了

- 不使用pool，那么对象占用内存越大，性能下降越厉害；使用pool，无论对象占用内存大还是小，性能都保持不变。可以看到pool有点像飞机，虽然起步比跑车慢，但后劲十足。

即：**pool适合占用内存大且并发量大的场景。当内存小并发量少的时候，使用pool适得其反**

**正确用法**

**在Put之前重置，在Get之后重置**

```

package main

import (
    "log"
    "runtime"
    "sync"
)

func main() {
    p := &sync.Pool{
        New: func() interface{} {
            return 0
        },
    }
    a := p.Get().(int)
    p.Put(6)
    b := p.Get().(int)
    log.Println(a, b)
    p.Put(666)
    p.Put(66)
    p.Put(6666)
    log.Println(p.Get()) //返回 66 666 666中的任意一个。
    //主动调用GC  pool中对象会被清理掉
    runtime.GC()
    log.Println(p.Get()) // 返回0， 之前对象被清理掉了

}

```

实例：gin的context通过pool来get和put，也就是使用了sync.Pool进行维护

```
// Conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := engine.pool.Get().(*Context) //pool的Get操作
    c.writermem.reset(w)
    c.Request = req
    c.reset() //重置
    engine.handleHTTPRequest(c)
    engine.pool.Put(c) //pool的Put操作
}

```


---


## **6.Once 执行一次**

<br/>

使用 sync.Once 对象可以使得函数多次调用只执行一次。其结构为：

```

type Once struct {
    m    Mutex
    done uint32
}

func (o *Once) Do(f func())

```

用 done 来记录执行次数，用 m 来保证保证仅被执行一次。只有一个 Do 方法，调用执行。

```
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(onceBody)
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}

# 打印结果
Only once

```