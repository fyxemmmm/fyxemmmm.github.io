---
title: 一文带你搞懂 golang channel 实现原理
date: 2021-09-03 00:11:55
author: yuxuan
img: https://image.fyxemmmm.cn/blog/fj/images/fj-2.jpg
top: true
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: 图解channel实现！ 其实channel并没有你想象中的那么难 ~ 
categories: golang
tags:
  - golang
  - channel
---
# Section1 channel使用实例

channel主要是为了实现go的并发特性，用于并发通信的，也就是在不同的协程单元goroutine之间同步通信。

下面主要从三个方面来讲解：

- make channel，主要也就是hchan的数据结构原型；

- 发送和接收数据时，goroutine会怎么调度；

- 设计思考；

## 1.1 make channel

我们创建channel时候有两种，一种是带缓冲的channel一种是不带缓冲的channel。创建方式分别如下：

```go
// buffered
ch := make(chan Task, 3)
// unbuffered
ch := make(chan int)
```

buffered channel

如果我们创建一个带buffer的channel，底层的数据模型如下图：

![](https://image.fyxemmmm.cn/blog/images/channel-0.png)
当我们向channel里面写入数据时候，会直接把数据存入circular queue(send)。当Queue存满了之后就会是如下的状态：
![](https://image.fyxemmmm.cn/blog/images/channel-1.png)

当dequeue一个元素时候，如下所示：
![](https://image.fyxemmmm.cn/blog/images/channel-2.png)

从上图可知，recvx自增加一，表示出队了一个元素，其实也就是循环数组实现FIFO语义。

那么还有一个问题，当我们新建channel的时候，底层创建的hchan数据结构是在哪里分配内存的呢？其实Section2里面源码分析时候已经做了分析，hchan是在heap里面分配的。

如下图所示：
![](https://image.fyxemmmm.cn/blog/images/channel-3.png)

当我们使用make去创建一个channel的时候，实际上返回的是一个指向channel的pointer，所以我们能够在不同的function之间直接传递channel对象，而不用通过指向channel的指针。

## 1.2 sends and receives

不同goroutine在channel上面进行读写时，涉及到的过程比较复杂，比如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-4.png)

上图中G1会往channel里面写入数据，G2会从channel里面读取数据。

G1作用于底层hchan的流程如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-5.png)

1. 先获取全局锁；
2. 然后enqueue元素(通过移动拷贝的方式)；
3. 释放锁；



G2读取时候作用于底层数据结构流程如下图所示：
![](https://image.fyxemmmm.cn/blog/images/channel-6.png)

1. 先获取全局锁；
2. 然后dequeue元素(通过移动拷贝的方式)；
3. 释放锁；

上面的读写思路其实很简单，除了hchan数据结构外，不要通过共享内存去通信；而是通过通信(复制)实现共享内存。

写入满channel的场景

如下图所示：channel写入3个task之后队列已经满了，这时候G1再写入第四个task的时候会发生什么呢？
![](https://image.fyxemmmm.cn/blog/images/channel-7.png)

G1这时候会暂停直到出现一个receiver。

这个地方需要介绍一下Golang的scheduler的。我们知道goroutine是用户空间的线程，创建和管理协程都是通过Go的runtime，而不是通过OS的thread。

但是Go的runtime调度执行goroutine却是基于OS thread的。如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-8.png)



具体关于golang的scheduler的原理，可以看前面的一篇博客，关于go的scheduler原理分析。

当向已经满的channel里面写入数据时候，会发生什么呢？如下图：

![](https://image.fyxemmmm.cn/blog/images/channel-9.png)

上图流程大概如下：

1. 当前goroutine（G1）会调用gopark函数，将当前协程置为waiting状态；
2. 将M和G1绑定关系断开；
3. scheduler会调度另外一个就绪态的goroutine与M建立绑定关系，然后M 会运行另外一个G。

所以整个过程中，OS thread会一直处于运行状态，不会因为协程G1的阻塞而阻塞。最后当前的G1的引用会存入channel的sender队列(队列元素是持有G1的sudog)。

那么blocked的G1怎么恢复呢？当有一个receiver接收channel数据的时候，会恢复 G1。

实际上hchan数据结构也存储了channel的sender和receiver的等待队列。数据原型如下：
![](https://image.fyxemmmm.cn/blog/images/channel-10.png)

等待队列里面是sudog的单链表，sudog持有一个G代表goroutine对象引用，elem代表channel里面保存的元素。当G1执行`ch<-task4`的时候，G1会创建一个sudog然后保存进入sendq队列，实际上hchan结构如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-11.png)

这个时候，如果G1进行一个读取channel操作，读取前和读取后的变化图如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-12.png)

整个过程如下：

1. G2调用 `t:=<-ch` 获取一个元素；
2. 从channel的buffer里面取出一个元素task1；
3. 从sender等待队列里面pop一个sudog；
4. 将task4复制buffer中task1的位置，然后更新buffer的sendx和recvx索引值；
5. 这时候需要将G1置为Runable状态，表示G1可以恢复运行；

这个时候将G1恢复到可运行状态需要scheduler的参与。G2会调用goready(G1)来唤醒G1。流程如下图所示：
![](https://image.fyxemmmm.cn/blog/images/channel-13.png)

1. 首先G2会调用goready(G1)，唤起scheduler的调度；
2. 将G1设置成Runable状态；
3. G1会加入到局部调度器P的local queue队列，等待运行。

读取空channel的场景

当channel的buffer里面为空时，这时候如果G2首先发起了读取操作。如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-14.png)

会创建一个sudog，将代表G2的sudog存入recvq等待队列。然后G2会调用gopark函数进入等待状态，让出OS thread，然后G2进入阻塞态。

这个时候，如果有一个G1执行读取操作，最直观的流程就是：

1. 将recvq中的task存入buffer；
2. goready(G2) 唤醒G2；

但是我们有更加智能的方法：direct send; 其实也就是G1直接把数据写入到G2中的elem中，这样就不用走G2中的elem复制到buffer中，再从buffer复制给G1。如下图：
![](https://image.fyxemmmm.cn/blog/images/channel-15.png)

具体过程就是G1直接把数据写入到G2的栈中。这样 G2 不需要去获取channel的全局锁和操作缓冲。

## 1.3 channel主要特性

（1）goroutine-safe
hchan mutex

（2）store values, pass in FIFO.
copying into and out of hchan buffer

（3）can cause goroutines to pause and resume.
a）hchan sudog queues

b）calls into the runtime scheduler (gopark, goready)

（4）channel的高性能所在：
a）调用runtime scheduler实现，OS thread不需要阻塞；
b）跨goroutine栈可以直接进行读写；

# Section2 源码分析

## 2.1 channel数据存储结构

在源码`runtime/chan.go` 里面定义了channel的数据模型，channel可以理解成一个缓冲队列，这个缓冲队列用来存储元素，并且提供FIFO的语义。源码如下：

```go
type hchan struct {
	//channel队列里面总的数据量
	qcount   uint           // total data in the queue
	// 循环队列的容量，如果是非缓冲的channel就是0
	dataqsiz uint           // size of the circular queue
	// 缓冲队列，数组类型。
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	// 元素占用字节的size
	elemsize uint16
	// 当前队列关闭标志位，非零表示关闭
	closed   uint32
	// 队列里面元素类型
	elemtype *_type // element type
	// 队列send索引
	sendx    uint   // send index
	// 队列索引
	recvx    uint   // receive index
	// 等待channel的G队列。
	recvq    waitq  // list of recv waiters
	// 向channel发送数据的G队列。
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	// 全局锁
	lock mutex
}
```

channel的数据结构相对比较简单，主要是两个结构：
1）一个数组实现的环形队列，数组有两个下标索引分别表示读写的索引，用于保存channel缓冲区数据。
2）channel的send和recv队列，队列里面都是持有goroutine的sudog元素，队列都是双链表实现的。
3）channel的全局锁。

## 2.2 make channel

我们新建一个channel的时候一般使用 `make(chan, n)` 语句，这个语句的执行编译器会重写然后执行 chan.go里面的 makechan函数。函数源码如下：

```go
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
		c.buf = unsafe.Pointer(c)
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

函数接收两个参数，一个是channel里面保存的元素的数据类型，一个是缓冲的容量(如果为0表示是非缓冲buffer)，创建流程如下：

- 根据传递的缓冲大小size是否为零，分别创建不带buffer的channel或则带size大小的缓冲channel：
  - 对于不带缓冲channel，申请一个hchan数据结构的内存大小；
  - 对于带缓冲channel，new一个hchan对象，并初始化buffer内存；
- 更新 chan中循环队列的关键属性：elemsize、elemtype、dataqsiz。

所以，整个创建channel的过程还是比较简单的。

## 2.3 协程从channel接收数据(goroutine receive data)

所有执行 `ep < c` 使用ep接收channel数据的代码，最后都会调用到chan.go里面的 `chanrecv函数`。

函数的定义如下：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
......
}
```

从源码注释就可以知道，该函数从channel里面接收数据，然后将接收到的数据写入到ep指针指向的对象里面。

还有一个参数block，表示当channel无法返回数据时是否阻塞等待。当block=false并且channel里面没有数据时候，函数直接返回(false,false)。

接收channel的数据的流程如下：

- CASE1：前置channel为nil的场景：
  - 如果block为非阻塞，直接return；
  - 如果block为阻塞，就调用gopark()阻塞当前goroutine，并抛出异常。
- 前置场景，block为非阻塞，且channel为非缓冲队列且sender等待队列为空 或则 channel为有缓冲队列但是队列里面元素数量为0，且channel未关闭，这个时候直接return；
- 调用 `lock(&c.lock)` 锁住channel的全局锁；
- CASE2：channel已经被关闭且channel缓冲中没有数据了，这时直接返回success和空值；
- CASE3：sender队列非空，调用`func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int)` 函数处理：
  - channel是非缓冲channel，直接调用recvDirect函数直接从sender recv元素到ep对象，这样就只用复制一次；
  - 对于sender队列非空情况下， 有缓冲的channel的缓冲队列一定是满的：
    - 1.先取channel缓冲队列的对头元素复制给receiver(也就是ep)；
    - 2.将sender队列的对头元素里面的数据复制到channel缓冲队列刚刚弹出的元素的位置，这样缓冲队列就不用移动数据了。
  - 释放channel的全局锁；
  - 调用goready函数标记当前goroutine处于ready，可以运行的状态；
- CASE4：sender队列为空，缓冲队列非空，直接取队列元素，移动头索引；
- CASE5：sender队列为空、缓冲队列也没有元素且不阻塞协程，直接return (false,false)；
- CASE6：sender队列为空且channel的缓存队列为空，将goroutine加入recv队列，并阻塞。

## 2.4 协程向channel写入数据(goroutine sender data)

所有执行 `c < ep` 将ep发送到channel的代码，最后都会调用到chan.go里面的 `chansend函数`。

函数的定义如下：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
......
}
```

函数有三个参数，第一个代表channel的数据结构，第二个是要指向写入的数据的指针，第三个block代表写入操作是否阻塞。

向channel写入数据主要流程如下：

- CASE1：当channel为空或者未初始化，如果block表示阻塞那么向其中发送数据将会永久阻塞；如果block表示非阻塞就会直接return；
- CASE2：前置场景，block为非阻塞，且channel没有关闭(已关闭的channel不能写入数据)且(channel为非缓冲队列且receiver等待队列为空)或则( channel为有缓冲队列但是队列已满)，这个时候直接return；
- 调用 `lock(&c.lock)` 锁住channel的全局锁；
- CASE3：不能向已经关闭的channel send数据，会导致panic。
- CASE4：如果channel上的recv队列非空，则跳过channel的缓存队列，直接向消息发送给接收的goroutine：
  - 调用sendDirect方法，将待写入的消息发送给接收的goroutine；
  - 释放channel的全局锁；
  - 调用goready函数，将接收消息的goroutine设置成就绪状态，等待调度。
- CASE5：缓存队列未满，则将消息复制到缓存队列上，然后释放全局锁；
- CASE6：缓存队列已满且接收消息队列recv为空，则将当前的goroutine加入到send队列
  - 获取当前goroutine的sudog，然后入channel的send队列；
  - 将当前goroutine休眠

## 2.5 channel close关闭channel源码分析

当我们执行channel的close操作的时候会关闭channel。

关闭的主要流程如下所示：

- 获取全局锁；
- 设置channel数据结构chan的关闭标志位；
- 获取当前channel上面的读goroutine并链接成链表；
- 获取当前channel上面的写goroutine然后拼接到前面的读链表后面；
- 释放全局锁；
- 唤醒所有的读写goroutine。