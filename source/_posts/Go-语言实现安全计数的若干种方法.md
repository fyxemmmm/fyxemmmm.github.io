---
title: golang 安全计数
date: 2021-09-19 13:04:54
img: https://image.fyxemmmm.cn/blog/images/ins-170.jpg
top: false
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: Go 语言实现安全计数的若干种方法
categories: golang
tags:
  - 安全计数
---

本文是一篇如何用 goroutine-safe 的方式实现计数器的方法汇总。

### 不要这样做

我们先从非安全的实现方式开始：

```go
type NotSafeCounter struct {
 number uint64
}

func NewNotSafeCounter() Counter {
 return &NotSafeCounter{0}
}

func (c *NotSafeCounter) Add(num uint64) {
 c.number = c.number + num
}

func (c *NotSafeCounter) Read() uint64 {
 return c.number
}
```

代码上没什么特别的地方。我们来测试下结果正确与否：创建 100 个 goroutine，其中三分之二的 goroutine 对共享计数器加一。

```go
func testCorrectness(t *testing.T, counter Counter) {
 wg := &sync.WaitGroup{}
 for i := 0; i < 100; i++ {
  wg.Add(1)
  if i%3 == 0 {
   go func(counter Counter) {
    counter.Read()
    wg.Done()
   }(counter)
  } else if i%3 == 1 {
   go func(counter Counter) {
    counter.Add(1)
    counter.Read()
    wg.Done()
   }(counter)
  } else {
   go func(counter Counter) {
    counter.Add(1)
    wg.Done()
   }(counter)
  }
 }

 wg.Wait()

 if counter.Read() != 66 {
  t.Errorf("counter should be %d and was %d", 66, counter.Read())
 }
}
```

测试的结果是不确定的，有时候能正确运行，有时候会出现类似这样的错误：

```
counter_test.go:34: counter should be 66 and was 65
```

### 经典实现方式

实现一个正确计数器的传统方式是使用**互斥锁**，保证任意时间只有一个协程操作计数器。Go 语言的话，我们可以使用 sync 包。

```go
type MutexCounter struct {
 mu     *sync.RWMutex
 number uint64
}

func NewMutexCounter() Counter {
 return &MutexCounter{&sync.RWMutex{}, 0}
}

func (c *MutexCounter) Add(num uint64) {
 c.mu.Lock()
 defer c.mu.Unlock()
 c.number = c.number + num
}

func (c *MutexCounter) Read() uint64 {
 c.mu.RLock()
 defer c.mu.RUnlock()
 return c.number
}
```

现在测试结果每次都能通过且都是正确的。

### 使用 channel

锁是一种保证同步的低级原语。Go 也提供了更高级实现方式 - channel。

关于 mutexe 和 channel，现在有太多类似这样的讨论：“mutexe vs channel ”、“哪个更好”、“我应当使用哪一个”等。其中一些讨论非常有趣且有益，但这并不是本文讨论的重点。

我们使用 channel 来实现协程安全的计数器，使用 channel 充当队列，对计数器的操作(读、写)都缓存在队列中，按顺序操作。具体的操作通过传递 func() 实现。创建时，计数器会衍生出一个 goroutine 并且按顺序执行队列里的操作。

下面是计数器的定义：

```go
type ChannelCounter struct {
 ch     chan func()
 number uint64
}

func NewChannelCounter() Counter {
 counter := &ChannelCounter{make(chan func(), 100), 0}
 go func(counter *ChannelCounter) {
  for f := range counter.ch {
   f()
  }
 }(counter)
 return counter
}
```

当一个协程调用 Add()，就往队列里面添加一个写操作：

```go
func (c *ChannelCounter) Add(num uint64) {
 c.ch <- func() {
  c.number = c.number + num
 }
}
```

当一个协程调用 Read()，就往队列里面添加一个读操作：

```go
func (c *ChannelCounter) Read() uint64 {
 ret := make(chan uint64)
 c.ch <- func() {
  ret <- c.number
  close(ret)
 }
 return <-ret
}
```

我真正喜欢这个实现的地方在于，这种按顺序执行的方式非常的清晰。

### 原子方式

我们甚至可以用更低级别的原语，利用 sync/atomic 包执行原子操作。

```go
type AtomicCounter struct {
 number uint64
}

func NewAtomicCounter() Counter {
 return &AtomicCounter{0}
}

func (c *AtomicCounter) Add(num uint64) {
 atomic.AddUint64(&c.number, num)
}

func (c *AtomicCounter) Read() uint64 {
 return atomic.LoadUint64(&c.number)
}
```

### 比较和交换

或者，我们可以使用非常经典的原语：CAS，对计时器进行计数。

```go
func (c *CASCounter) Add(num uint64) {
 for {
  v := atomic.LoadUint64(&c.number)
  if atomic.CompareAndSwapUint64(&c.number, v, v+num) {
   return
  }
 }
}

func (c *CASCounter) Read() uint64 {
 return atomic.LoadUint64(&c.number)
}
```

### float 类型该如何实现

在我探索学习过程中，看到一个非常棒的视频 - 《**Prometheus: Designing and Implementing a Modern Monitoring Solution in Go**[1]》。在视频的最后，讨论了如何实现浮点数计数器。到目前为止，所有的技术都适用于浮点数，除了 sync/atomic 包，还没提供浮点数的原子操作。

在视频里，Björn Rabenstein 介绍了如何通过将浮点数存储为 uint64 并使用 math.Float64bits 和 math.Float64frombits 在 float64 和 uint64 之间进行转换来解决此问题。

```go
type CASFloatCounter struct {
 number uint64
}

func NewCASFloatCounter() *CASFloatCounter {
 return &CASFloatCounter{0}
}

func (c *CASFloatCounter) Add(num float64) {
 for {
  v := atomic.LoadUint64(&c.number)
  newValue := math.Float64bits(math.Float64frombits(v) + num)
  if atomic.CompareAndSwapUint64(&c.number, v, newValue) {
   return
  }
 }
}

func (c *CASFloatCounter) Read() float64 {
 return math.Float64frombits(atomic.LoadUint64(&c.number))
}
```

### 最后

这篇文章是共享计数器的实现汇总。这是我好奇心驱使的结果，此外对并发也有一个基本的了解。