---
title: Linux syscall 原理
date: 2021-09-30 22:00:45
img: https://image.fyxemmmm.cn/blog/images/ins-56.jpg
top: false
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: syscall调用流程 の 简述
categories: 操作系统
tags:
  - 系统调用
---
## 一、Syscall意义

内核提供用户空间程序与内核空间进行交互的一套标准接口，这些接口让用户态程序能受限访问硬件设备，比如申请系统资源，操作设备读写，创建新进程等。用户空间发生请求，内核空间负责执行，这些接口便是用户空间和内核空间共同识别的桥梁，这里提到两个字“受限”，是由于为了保证内核稳定性，而不能让用户空间程序随意更改系统，必须是内核对外开放的且满足权限的程序才能调用相应接口。

在用户空间和内核空间之间，有一个叫做Syscall(系统调用, system call)的中间层，是连接用户态和内核态的桥梁。这样即提高了内核的安全型，也便于移植，只需实现同一套接口即可。Linux系统，用户空间通过向内核空间发出Syscall，产生软中断，从而让程序陷入内核态，执行相应的操作。对于每个系统调用都会有一个对应的系统调用号，比很多操作系统要少很多。

安全性与稳定性：内核驻留在受保护的地址空间，用户空间程序无法直接执行内核代码，也无法访问内核数据，通过系统调用

性能：Linux上下文切换时间很短，以及系统调用处理过程非常精简，内核优化得好，所以性能上往往比很多其他操作系统执行要好。

## 二、Syscall查找方式

这里以kill()方法为例子，来找一找kill()方法系统调用的过程。

Tips 1： 用户空间的方法`xxx`，对应系统调用层方法则是`sys_xxx`； 
Tips 2： `unistd.h`文件记录着系统调用中断号的信息。

故用户空间`kill`方法则对应系统调用层便是`sys_kill`，这个方法去哪里找呢？从`/kernel/include/uapi/asm-generic/unistd.h`等还有很多`unistd.h`去慢慢查看，查看关键字`sys_kill`，便能看到下面几行：

```c
/* kernel/signal.c */
__SYSCALL(__NR_kill, sys_kill)
```

根据这个能得到一丝线索，那就是kill对应的方法sys_kill位于`/kernel/signal.c`文件。

Tips 3： 宏定义SYSCALL_DEFINEx(xxx,…)，展开后对应的方法则是`sys_xxx`； 
Tips 4： 方法参数的个数x，对应于SYSCALL_DEFINEx。

`kill(int pid, int sig)`方法共两个参数，则对应方法于`SYSCALL_DEFINE2(kill,...)`，进入signal.c文件，再次搜索关键字，便能看到方法：

```c
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
    struct siginfo info;
    info.si_signo = sig;
    info.si_errno = 0;
    info.si_code = SI_USER;
    info.si_pid = task_tgid_vnr(current);
    info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
    return kill_something_info(sig, &info, pid);
}
```

`SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)` 基本等价于 `asmlinkage long sys_kill(int pid, int sig)`，这里用的是基本等价，往下看会解释原因。

#### 实用技巧

比如kill命令, 有两个参数. 则可以直接在kernel目录下搜索 “SYSCALL_DEFINE2(kill”,即可直接找到,所有对应的Syscall方法位于signal.c

## 三、Syscall流程

Syscall是通过中断方式实现的，ARM平台上通过swi中断来实现系统调用，实现从用户态切换到内核态，发送软中断swi时，从中断向量表中查看跳转代码，其中异常向量表定义在文件/kernelarch/arm/kernel/entry-armv.S(汇编语言文件)。当执行系统调用时会根据系统调用号从系统调用表中来查看目标函数的入口地址，在calls.S文件中声明了入口地址信息。

总体流程：kill() -> kill.S -> swi陷入内核态 -> 从sys_call_table查看到sys_kill -> ret_fast_syscall -> 回到用户态执行kill()下一行代码。 下面介绍部分核心流程：

3.1： 用户程序通过软中断swi指令切入内核态，执行vector_swi处的指令。`vector_swi`在文件`/kenel/arch/arm/kernel/entry-common.S`中定义，此处省略。像每一个异常处理程序一样，要做的第一件事当然就是保护现场了。紧接着是获得系统调用的系统调用号

3.2： 仍以kill()函数为例，来详细说说Syscall调用流程，用户空间kill()定义位于文件`kill.S`。

```c
#include <private/bionic_asm.h>
ENTRY(kill)
    mov     ip, r7
    ldr     r7, =__NR_kill
    swi     #0
    mov     r7, ip
    cmn     r0, #(MAX_ERRNO + 1)
    bxls    lr
    neg     r0, r0
    b       __set_errno_internal
END(kill)
```

当调用kill时, 系统先保存r7内容, 然后将__NR_kill值放入r7, 再执行swi软中断指令切换进内核态。

3.3： Linux内核中，每个Syscall都有唯一的系统调用号对应，kill的系统调用号为__NR_kill，用户空间的系统调用号定义于`/bionic/libc/kernel/uapi/asm-generic/unistd.h`，如下：

```c
#define __NR_kill (__NR_SYSCALL_BASE + 37)
```

其中__NR_SYSCALL_BASE=0，也就是__NR_kill系统调用号=37。

3.4： 在内核中有与系统调用号对应的系统调用表，定义在文件`/kernel/arch/arm/kernel/calls.S`，如下：

```c
/* 35 */    CALL(sys_ni_syscall)        /* was sys_ftime */
            CALL(sys_sync)
            CALL(sys_kill)  //此处为37号
            CALL(sys_rename)
            CALL(sys_mkdir)
```

到这里可知37号系统调用对应sys_kill()，该方法所对应的函数声明在syscalls.h文件

3.5： 文件`/kernel/include/linux/syscalls.h`中有如下声明：

```c
asmlinkage long sys_kill(int pid, int sig);
```

asmlinkage是gcc标签，代表函数读取的参数来自于栈中，而非寄存器。

#### 3.1 SYSCALL_DEFINE

sys_kill()定义在内核源码找不到直接定义，而是通过`syscalls.h`文件中的SYSCALL_DEFINE宏定义。前面已经讲过sys_kill是通过语句`SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)`来定义，下面来一层层剖开，这条宏定义的真面目：

等价 1：

`syscalls.h`中有大量如下宏定义：

```c
#define SYSCALL_DEFINE0(sname)                    \
    SYSCALL_METADATA(_##sname, 0);                \
    asmlinkage long sys_##sname(void)
#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)
```

可得出原语句等价：

```c
SYSCALL_DEFINEx(2, _kill, pid_t, pid, int, sig)
```

等价 2：

`syscalls.h`中有如下宏定义：

```c
#define SYSCALL_DEFINEx(x, sname, ...)                \
    SYSCALL_METADATA(sname, x, __VA_ARGS__)            \
    __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

可得出原语句等价：

```c
SYSCALL_METADATA(_kill, 2, pid_t, pid, int, sig)
__SYSCALL_DEFINEx(2, _kill, pid_t, pid, int, sig)
```

define __SYSCALL_DEFINEx(x, name, …)

等价 3：

`syscalls.h`中有如下宏定义：

```c
#define __SYSCALL_DEFINEx(x, name, ...)                    \
    asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))    \
        __attribute__((alias(__stringify(SyS##name))));        \
    static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));    \
    asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));    \
    asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))    \
    {                                \
        long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));    \
        __MAP(x,__SC_TEST,__VA_ARGS__);                \
        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));    \
        return ret;                        \
    }                                \
    static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

可得出原语句等价：

```c
asmlinkage long sys_kill(__MAP(2,__SC_DECL,__VA_ARGS__))    \
    __attribute__((alias(__stringify(SyS_kill))));        \
static inline long SYSC_kill(__MAP(2,__SC_DECL,__VA_ARGS__));    \
asmlinkage long SyS_kill(__MAP(2,__SC_LONG,__VA_ARGS__));    \
asmlinkage long SyS_kill(__MAP(2,__SC_LONG,__VA_ARGS__))    \
{                                \
    long ret = SYSC_kill(__MAP(2,__SC_CAST,__VA_ARGS__));    \
    __MAP(2,__SC_TEST,__VA_ARGS__);                \
    __PROTECT(2, ret,__MAP(2,__SC_ARGS,__VA_ARGS__));    \
    return ret;                        \
}                                \
static inline long SYSC_kill(__MAP(2,__SC_DECL,__VA_ARGS__))
```

这里`__VA_ARGS__`等于 `pid_t, pid, int, sig`。

等价 4:

先说说这里涉及的宏定义

__MAP宏定义：

```c
#define __MAP0(m,...)
#define __MAP1(m,t,a) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)
```

相关宏定义：

```c
#define __SC_DECL(t, a)    t a
#define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
#define __SC_CAST(t, a)    (t) a
#define __SC_ARGS(t, a)    a
#define __SC_TEST(t, a) (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(t) && sizeof(t) > sizeof(long))
```

展开：

```c
__MAP(2,__SC_DECL, pid_t, pid, int, sig) //等价于 pid_t pid, int sig
__MAP(2,__SC_LONG,__VA_ARGS__) //等价于 long pid, long sig
__MAP(2,__SC_CAST,__VA_ARGS__) //等价于 (pid_t) pid, (int)sig
__MAP(2,__SC_ARGS,__VA_ARGS__) //等价于 pid, sig
```

可得出原语句等价：

```c
//函数声明sys_kill()，并别名指向SyS_kill
asmlinkage long sys_kill(pid_t pid, int sig) __attribute__((alias(__stringify(SyS_kill))));
static inline long SYSC_kill(pid_t pid, int sig);

//函数声明SyS_kill()
asmlinkage long SyS_kill(long pid, long sig);
asmlinkage long SyS_kill(long pid, long sig)
{
    long ret = SYSC_kill((pid_t) pid, (int)sig);
    BUILD_BUG_ON_ZERO(sizeof(pid_t) > sizeof(long));
    BUILD_BUG_ON_ZERO(sizeof(int) > sizeof(long));
    __PROTECT(2, ret, pid, sig);
    return ret;
}
static inline long SYSC_kill(pid_t pid, int sig)
```

通过以上分析过程：

- kill添加了`sys_`前缀，声明sys_kill()函数；
- 定义SYSC_kill()函数和SyS_kill()函数；
- sys_kill，通过别名机制等同于SyS_kill().

看到这或许很多人(包括我)会觉得诧异，为何要如此复杂呢，后来查资料，发现这是由于之前64位Linux存在`CVE-2009-2009`的漏洞，简单说就是32位参数存放在64位寄存器，修改符号扩展可能导致产生一个非法内存地址，从而导致系统崩溃或者提升权限。 为了修复这个问题，把寄存器高位清零即可，但做起来比较困难，为了做尽可能少的修改，将调用参数统一采用使用long型来接收，再强转为相应参数。 窥见一斑，可见Linux大师们精湛的宏定义，已经用得出神入化。

如果觉得很复杂，那么可以忽略这个宏定义，只要记住`SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)` 基本等价于 `asmlinkage long sys_kill(int pid, int sig)` 就足够了。

## 四、总结

#### 4.1 内核空间

1. 系统调用的函数原型的指针：位于文件/kernel/arch/arm/kernel/calls.S，格式为`CALL(sys_xxx)`，指定了目标函数的入口地址。
2. 系统调用号的宏定义：位于文件/kernel/arch/arm/include/Uapi/asm/unistd.h，记录着内核空间的系统调用号，格式为`#define__NR_xxx (__NR_SYSCALL_BASE+[num])`
3. 系统调用的函数声明：位于文件/kernel/include/linux/syscalls.h，格式为`asmlinkage long sys_xxx(args ...);`
4. 系统调用的函数实现：不同函数位于不同文件，比如kill()位于/kernel/kernel/signal.c文件，格式为`SYSCALL_DEFINEx(x, sname, ...)`

前面这4步都是在内核空间相关的文件定义，有了这些，那么内核就可以使用相应的系统调用号。

#### 4.2 用户空间

1. 系统调用号的宏定义：位于文件/bionic/libc/kernel/uapi/asm-arm/asm/unistd.h，记录着用户空间的系统调用号，格式为`#define__NR_xxx (__NR_SYSCALL_BASE+[num])`。这个文件就是由内核空间同名的头文件自动生成的，所以该文件与内核空间的系统调用号是完全一致。

2. 汇编定义相关函数的中断调用过程：位于文件/bionic/libc/arch-arm/syscalls/xxx.S，比如kill()位于kill.S，格式为：

   ```c
    ENTRY(xxx)
        mov     ip, r7
        ldr     r7, =__NR_xxx
        swi     #0
        mov     r7, ip
        cmn     r0, #(MAX_ERRNO + 1)
        bxls    lr
        neg     r0, r0
        b       __set_errno_internal
    END(xxx)
   ```

当然kill()方法还有函数声明，有了这些，用户空间也能在程序中使用系统调用。明白了这些过程，那么自己新添加系统调用其实也并不是多困难的一件事，新增系统调用号还需要修改syscalls总个数，但强烈不建议自己新增系统调用号，尽量保持与linux kernel主线一致，兼容性更好，所以就不进一步介绍新增流程了。

 