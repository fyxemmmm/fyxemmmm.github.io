---
title: golang [ 逃逸分析 ]
date: 2021-09-02 00:46:00
author: yuxuan
img: https://image.fyxemmmm.cn/blog/images/ins-31.jpg
top: false
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: golang的内存逃逸情况是什么样的呢？
categories: golang
tags:
  - golang
  - 内存管理
---
## 介绍

Go 语言较之 C 语言一个很大的优势就是自带 GC 功能，可 GC 并不是没有代价的。写 C 语言的时候，在一个函数内声明的变量，在函数退出后会自动释放掉，因为这些变量分配在栈上。如果你想要变量的数据能在函数退出后还能访问，就需要调用 `malloc` 方法在堆上申请内存，如果程序不再需要这块内存了，再调用 `free` 方法释放掉。Go 语言不需要你主动调用 `malloc` 来分配堆空间，编译器会自动分析，找出需要 `malloc` 的变量，使用堆内存。编译器的这个分析过程就叫做逃逸分析。

所以你在一个函数中通过 `dict := make(map[string]int)` 创建一个 map 变量，其背后的数据是放在栈空间上还是堆空间上，是不一定的。这要看编译器分析的结果。

可逃逸分析并不是百分百准确的，它有缺陷。有的时候你会发现有些变量其实在栈空间上分配完全没问题的，但编译后程序还是把这些数据放在了堆上。如果你了解 Go 语言编译器逃逸分析的机制，在写代码的时候就可以有意识的绕开这些缺陷，使你的程序更高效。

## 关于堆栈

Go 语言虽然在内存管理方面降低了编程门槛，即使你不了解堆栈也能正常开发，但如果你要在性能上较真的话，还是要掌握这些基础知识。

这里不对堆内存和栈内存的区别做太多阐述。简单来说就是，**栈分配廉价，堆分配昂贵。**栈空间会随着一个函数的结束自动释放，堆空间需要 GC 模块不断的跟踪扫描回收。如果对这两个概念有些迷糊，建议阅读下面 ２ 个文章：

- [Language Mechanics On Stacks And Pointers](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.ardanlabs.com%2Fblog%2F2017%2F05%2Flanguage-mechanics-on-stacks-and-pointers.html)
- [Language Mechanics On Escape Analysis](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.ardanlabs.com%2Fblog%2F2017%2F05%2Flanguage-mechanics-on-escape-analysis.html)

这里举一个小例子，来对比下堆栈的差别：



```go
func stack() int { 
    // 变量 i 会在栈上分配
     i := 10
     return i
}
func heap() *int {
    // 变量 j 会在堆上分配
    j := 10
    return &j
}
```

`stack` 函数中的变量 `i` 在函数退出会自动释放；而 `heap` 函数返回的是对变量`i`的引用，也就是说 `heap()`退出后，表示变量 `i`  还要能被访问，它会自动被分配到堆空间上。

他们编译出来的代码如下：



```asm
// go build --gcflags '-l' test.go
// go tool objdump ./test


TEXT main.stack(SB) /tmp/test.go
  test.go:7     0x487240        48c74424080a000000  MOVQ $0xa, 0x8(SP)  
  test.go:7     0x487249        c3          RET         

TEXT main.heap(SB) /tmp/test.go
  test.go:9     0x487250        64488b0c25f8ffffff  MOVQ FS:0xfffffff8, CX          
  test.go:9     0x487259        483b6110        CMPQ 0x10(CX), SP           
  test.go:9     0x48725d        7639            JBE 0x487298                
  test.go:9     0x48725f        4883ec18        SUBQ $0x18, SP              
  test.go:9     0x487263        48896c2410      MOVQ BP, 0x10(SP)           
  test.go:9     0x487268        488d6c2410      LEAQ 0x10(SP), BP           
  test.go:10        0x48726d        488d05ac090100      LEAQ 0x109ac(IP), AX            
  test.go:10        0x487274        48890424        MOVQ AX, 0(SP)              
  test.go:10        0x487278        e8f33df8ff      CALL runtime.newobject(SB)      
  test.go:10        0x48727d        488b442408      MOVQ 0x8(SP), AX            
  test.go:10        0x487282        48c7000a000000      MOVQ $0xa, 0(AX)            
  test.go:11        0x487289        4889442420      MOVQ AX, 0x20(SP)           
  test.go:11        0x48728e        488b6c2410      MOVQ 0x10(SP), BP           
  test.go:11        0x487293        4883c418        ADDQ $0x18, SP              
  test.go:11        0x487297        c3          RET                 
  test.go:9     0x487298        e8a380fcff      CALL runtime.morestack_noctxt(SB)   
  test.go:9     0x48729d        ebb1            JMP main.heap(SB)           
// ...

TEXT runtime.newobject(SB) /usr/share/go/src/runtime/malloc.go
  malloc.go:1067    0x40b070        64488b0c25f8ffffff  MOVQ FS:0xfffffff8, CX          
  malloc.go:1067    0x40b079        483b6110        CMPQ 0x10(CX), SP           
  malloc.go:1067    0x40b07d        763d            JBE 0x40b0bc                
  malloc.go:1067    0x40b07f        4883ec28        SUBQ $0x28, SP              
  malloc.go:1067    0x40b083        48896c2420      MOVQ BP, 0x20(SP)           
  malloc.go:1067    0x40b088        488d6c2420      LEAQ 0x20(SP), BP           
  malloc.go:1068    0x40b08d        488b442430      MOVQ 0x30(SP), AX           
  malloc.go:1068    0x40b092        488b08          MOVQ 0(AX), CX              
  malloc.go:1068    0x40b095        48890c24        MOVQ CX, 0(SP)              
  malloc.go:1068    0x40b099        4889442408      MOVQ AX, 0x8(SP)            
  malloc.go:1068    0x40b09e        c644241001      MOVB $0x1, 0x10(SP)         
  malloc.go:1068    0x40b0a3        e888f4ffff      CALL runtime.mallocgc(SB)       
  malloc.go:1068    0x40b0a8        488b442418      MOVQ 0x18(SP), AX           
  malloc.go:1068    0x40b0ad        4889442438      MOVQ AX, 0x38(SP)           
  malloc.go:1068    0x40b0b2        488b6c2420      MOVQ 0x20(SP), BP           
  malloc.go:1068    0x40b0b7        4883c428        ADDQ $0x28, SP              
  malloc.go:1068    0x40b0bb        c3          RET                 
  malloc.go:1067    0x40b0bc        e87f420400      CALL runtime.morestack_noctxt(SB)   
  malloc.go:1067    0x40b0c1        ebad            JMP runtime.newobject(SB)       
```

逻辑的复杂度不言而喻，上面的汇编中可看到， `heap()` 函数调用了 `runtime.newobject()` 方法，它会调用 `mallocgc` 方法从 `mcache` 上申请内存，申请的内部逻辑前面文章已经讲述过。堆内存分配不仅分配上逻辑比栈空间分配复杂，它最致命的是会带来很大的管理成本，Go 语言要消耗很多的计算资源对其进行标记回收（也就是 GC 成本）。

> 不要以为使用了堆内存就一定会导致性能低下，使用栈内存会带来性能优势。因为实际项目中，系统的性能瓶颈一般都不会出现在内存分配上。千万不要盲目优化，找到系统瓶颈，用数据驱动优化。

## 逃逸分析

Go 编辑器会自动帮我们找出需要进行动态分配的变量，它是在编译时追踪一个变量的生命周期，如果能确认一个数据只在函数空间内访问，不会被外部使用，则使用栈空间，否则就要使用堆空间。

我们在 `go build` 编译代码时，可使用 `-gcflags '-m'` 参数来查看逃逸分析日志。



```sh
go build -gcflags '-m -m' test.go
```

以上面的两个函数为例，编译的日志输出是：



```go
/tmp/test.go:11:9: &i escapes to heap
/tmp/test.go:11:9:  from ~r0 (return) at /tmp/test.go:11:2
/tmp/test.go:10:2: moved to heap: i
/tmp/test.go:16:18: heap() escapes to heap
/tmp/test.go:16:18:     from ... argument (arg to ...) at /tmp/test.go:16:13
/tmp/test.go:16:18:     from *(... argument) (indirection) at /tmp/test.go:16:13
/tmp/test.go:16:18:     from ... argument (passed to call[argument content escapes]) at /tmp/test.go:16:13
/tmp/test.go:16:13: main ... argument does not escape
```

日志中的 `&i escapes to heap` 表示该变量数据逃逸到了堆上。

### 逃逸分析的缺陷

需要使用堆空间则逃逸，这没什么可争议的。但编译器有时会将**不需要**使用堆空间的变量，也逃逸掉。这里是容易出现性能问题的大坑。网上有很多相关文章，列举了一些导致逃逸情况，其实总结起来就一句话：

**多级间接赋值容易导致逃逸**。

这里的多级间接指的是，对某个引用类对象中的引用类成员进行赋值。Go 语言中的引用类数据类型有 `func`, `interface`, `slice`, `map`, `chan`, `*Type(指针)`。

记住公式 `Data.Field = Value`，如果 `Data`, `Field` 都是引用类的数据类型，则会导致 `Value` 逃逸。这里的等号 `=` 不单单只赋值，也表示参数传递。

根据公式，我们假设一个变量 `data` 是以下几种类型，相应的可得出结论：

- `[]interface{}`: `data[0] = 100` 会导致 `100` 逃逸
- `map[string]interface{}`: `data["key"] = "value"` 会导致 `"value"` 逃逸
- `map[interface{}]interface{}`: `data["key"] = "value"` 会导致 `key` 和 `value` 都逃逸
- `map[string][]string`: `data["key"] = []string{"hello"}` 会导致切片逃逸
- `map[string]*int`:  赋值时 `*int` 会 逃逸
- `[]*int`: `data[0] = &i` 会使 `i` 逃逸
- `func(*int)`: `data(&i)` 会使 `i` 逃逸
- `func([]string)`: `data([]{"hello"})` 会使 `[]string{"hello"}` 逃逸
- `chan []string`: `data <- []string{"hello"}` 会使 `[]string{"hello"}` 逃逸
- 以此类推，不一一列举了

下面给出一些实际的例子：

#### 函数变量

如果变量值是一个函数，函数的参数又是引用类型，则传递给它的参数都会逃逸。



```go
func test(i int)        {}
func testEscape(i *int) {}

func main() {
    i, j, m, n := 0, 0, 0, 0
    t, te := test, testEscape // 函数变量

    // 直接调用
    test(m)        // 不逃逸
    testEscape(&n) // 不逃逸
    // 间接调用
    t(i)   // 不逃逸
    te(&j) // 逃逸
}
```



```go
./test.go:4:17: testEscape i does not escape
./test.go:11:5: &j escapes to heap
./test.go:11:5:     from te(&j) (parameter to indirect call) at ./test.go:11:4
./test.go:7:5: moved to heap: j
./test.go:14:13: main &n does not escape
```

上例中 `te` 的类型是 `func(*int)`，属于引用类型，参数 `*int` 也是引用类型，则调用 `te(&j)` 形成了为 `te` 的参数(成员) `*int` 赋值的现象，即 `te.i = &j` 会导致逃逸。代码中其他几种调用都没有形成**多级间接赋值**情况。
 同理，如果函数的参数类型是 `slice`, `map` 或 `interface{}` 都会导致参数逃逸。



```go
func testSlice(slice []int)       {}
func testMap(m map[int]int)       {}
func testInterface(i interface{}) {}

func main() {
    x, y, z := make([]int, 1), make(map[int]int), 100
    ts, tm, ti := testSlice, testMap, testInterface
    ts(x) // ts.slice = x 导致 x 逃逸
    tm(y) // tm.m = y 导致 y 逃逸
    ti(z) // ti.i = z 导致 z 逃逸
}
```



```go
./test.go:3:16: testSlice slice does not escape
./test.go:4:14: testMap m does not escape
./test.go:5:20: testInterface i does not escape
./test.go:8:17: make([]int, 1) escapes to heap
./test.go:8:17:     from x (assign-pair) at ./test.go:8:10
./test.go:8:17:     from ts(x) (parameter to indirect call) at ./test.go:10:4
./test.go:8:33: make(map[int]int) escapes to heap
./test.go:8:33:     from y (assign-pair) at ./test.go:8:10
./test.go:8:33:     from tm(y) (parameter to indirect call) at ./test.go:11:4
./test.go:12:4: z escapes to heap
./test.go:12:4:     from ti(z) (parameter to indirect call) at ./test.go:12:4
```

匿名函数的调用也是一样的，它本质上也是一个函数变量。有兴趣的可以自己测试一下。

#### 间接赋值



```go
type Data struct {
    data  map[int]int
    slice []int
    ch    chan int
    inf   interface{}
    p     *int
}

func main() {
    d1 := Data{}
    d1.data = make(map[int]int) // GOOD: does not escape
    d1.slice = make([]int, 4)   // GOOD: does not escape
    d1.ch = make(chan int, 4)   // GOOD: does not escape
    d1.inf = 3                  // GOOD: does not escape
    d1.p = new(int)             //  GOOD: does not escape

    d2 := new(Data)             // d2 是指针变量， 下面为该指针变量中的指针成员赋值
    d2.data = make(map[int]int) // BAD: escape to heap
    d2.slice = make([]int, 4)   // BAD:  escape to heap
    d2.ch = make(chan int, 4)   // BAD:  escape to heap
    d2.inf = 3                  // BAD:  escape to heap
    d2.p = new(int)             // BAD:  escape to heap
}
```



```go
./test.go:20:16: make(map[int]int) escapes to heap
./test.go:20:16:    from d2.data (star-dot-equals) at ./test.go:20:10
./test.go:21:17: make([]int, 4) escapes to heap
./test.go:21:17:    from d2.slice (star-dot-equals) at ./test.go:21:11
./test.go:22:14: make(chan int, 4) escapes to heap
./test.go:22:14:    from d2.ch (star-dot-equals) at ./test.go:22:8
./test.go:23:9: 3 escapes to heap
./test.go:23:9:     from d2.inf (star-dot-equals) at ./test.go:23:9
./test.go:24:12: new(int) escapes to heap
./test.go:24:12:    from d2.p (star-dot-equals) at ./test.go:24:7
./test.go:13:16: main make(map[int]int) does not escape
./test.go:14:17: main make([]int, 4) does not escape
./test.go:15:14: main make(chan int, 4) does not escape
./test.go:16:9: main 3 does not escape
./test.go:17:12: main new(int) does not escape
./test.go:19:11: main new(Data) does not escape
```

#### Interface

只要使用了 `Interface` 类型(不是 `interafce{}`)，那么赋值给它的变量一定会逃逸。因为 `interfaceVariable.Method()` 先是间接的定位到它的实际值，再调用实际值的同名方法，执行时实际值作为参数传递给方法。相当于`interfaceVariable.Method.this = realValue`



```go
type Iface interface {
    Dummy()
}
type Integer int
func (i Integer) Dummy() {}

func main() {
    var (
        iface Iface
        i     Integer
    )
    iface = i
    iface.Dummy() //  make i escape to heap
    // 形成 iface.Dummy.i = i
}
```

#### 引用类型的 channel

向 channel 中发送数据，本质上就是为 channel 内部的成员赋值，就像给一个 slice 中的某一项赋值一样。所以 `chan *Type`, `chan map[Type]Type`, `chan []Type`, `chan interface{}` 类型都会导致发送到 channel 中的数据逃逸。

这本来也是情理之中的，发送给 channel 的数据是要与其他函数分享的，为了保证发送过去的指针依然可用，只能使用堆分配。



```go
func test() {
    var (
        chInteger   = make(chan *int)
        chMap       = make(chan map[int]int)
        chSlice     = make(chan []int)
        chInterface = make(chan interface{})
        a, b, c, d  = 0, map[int]int{}, []int{}, 32
    )
    chInteger <- &a  // 逃逸
    chMap <- b       // 逃逸
    chSlice <- c     // 逃逸
    chInterface <- d // 逃逸
}
```



```go
./escape.go:11:15: &a escapes to heap
./escape.go:11:15:  from chInteger <- &a (send) at ./escape.go:11:12
./escape.go:9:3: moved to heap: a
./escape.go:9:31: map[int]int literal escapes to heap
./escape.go:9:31:   from b (assigned) at ./escape.go:9:3
./escape.go:9:31:   from chMap <- b (send) at ./escape.go:12:8
./escape.go:9:40: []int literal escapes to heap
./escape.go:9:40:   from c (assigned) at ./escape.go:9:3
./escape.go:9:40:   from chSlice <- c (send) at ./escape.go:13:10
./escape.go:14:14: d escapes to heap
./escape.go:14:14:  from chInterface <- (interface {})(d) (send) at ./escape.go:14:14
./escape.go:5:21: test make(chan *int) does not escape
./escape.go:6:21: test make(chan map[int]int) does not escape
./escape.go:7:21: test make(chan []int) does not escape
./escape.go:8:21: test make(chan interface {}) does not escape
```

#### 可变参数

可变参数如 `func(arg ...string)` 实际与 `func(arg []string)` 是一样的，会增加一层访问路径。这也是 `fmt.Sprintf` 总是会使参数逃逸的原因。

例子非常多，这里不能一一列举，我们只需要记住分析方法就好，即，2 级或更多级的访问赋值会**容易**导致数据逃逸。这里加上**容易**二字是因为随着语言的发展，相信这些问题会被慢慢解决，但现阶段，这个可以作为我们分析逃逸现象的依据。

下面代码中包含 2 种很常规的写法，但他们却有着很大的性能差距，建议自己想下为什么。



```go
type User struct {
    roles []string
}

func (u *User) SetRoles(roles []string) {
    u.roles = roles
}

func SetRoles(u User, roles []string) User {
    u.roles = roles
    return u
}
```

Benchmark 和 pprof 给出的结果:



```go
BenchmarkUserSetRoles-8     50000000            22.3 ns/op        16 B/op          1 allocs/op
BenchmarkSetRoles-8         2000000000           0.51 ns/op        0 B/op          0 allocs/op

  768.01MB   768.01MB (flat, cum)   100% of Total
         .          .      3:import "testing"
         .          .      4:
         .          .      5:func BenchmarkUserSetRoles(b *testing.B) {
         .          .      6:   u := new(User)
         .          .      7:   for i := 0; i < b.N; i++ {
  768.01MB   768.01MB      8:       u.SetRoles([]string{"admin"}) <- 看这里
         .          .      9:   }
         .          .     10:}
         .          .     11:
         .          .     12:func BenchmarkSetRoles(b *testing.B) {
         .          .     13:   for i := 0; i < b.N; i++ {
ROUTINE ======================== testing.(*B).launch in /usr/share/go/src/testing/benchmark.go
......
```

## 结论

熟悉堆栈概念可以让我们更容易看透 Go 程序的性能问题，并进行优化。

多级间接赋值会导致 Go 编译器出现不必要的逃逸，在一些情况下可能我们只需要修改一下数据结构就会使性能有大幅提升。这也是很多人不推荐在 Go 中使用指针的原因，因为它会增加一级访问路径，而 `map`, `slice`, `interface{}`等类型是不可避免要用到的，为了减少不必要的逃逸，只能拿指针开刀了。

大多数情况下，性能优化都会为程序带来一定的复杂度。建议实际项目中还是怎么方便怎么写，功能完成后通过性能分析找到瓶颈所在，再对局部进行优化。


---
常见内存逃逸情况
1、在方法内把局部变量指针返回，被外部引用，其生命周期大于栈，则溢出。
2、发送指针或带有指针的值到channel，因为编译时候无法知道那个goroutine会在channel接受数据，编译器无法知道什么时候释放。
3、在一个切片上存储指针或带指针的值。比如[]*string，导致切片内容逃逸，其引用值一直在堆上。
4、因为切片的append导致超出容量，切片重新分配地址，切片背后的存储基于运行时的数据进行扩充，就会在堆上分配。
5、在interface类型上调用方法，在Interface调用方法是动态调度的，只有在运行时才知道。