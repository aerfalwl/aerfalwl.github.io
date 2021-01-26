---
title: C语言编译与汇编
date: 2021-01-26 11：17：42
categories: concurrent
tags:
	- concurrent
	- spinlock
	- test
---

# C语言编译与汇编

## 语法

### inline函数

```c
#define SPINLOCK_ATTR static __inline __attribute__((always_inline, no_instrument_function))
```

1. `__attribute__((always_inline))`的意思是强制内联，即加了该声明的函数在编译时不会编译成函数，而是直接扩展到调用函数体内。

## 汇编语法

```asm
asm asm-qualifiers ( AssemblerTemplate 
                 : OutputOperands 
                 [ : InputOperands
                 [ : Clobbers ] ])

asm asm-qualifiers ( AssemblerTemplate 
                      : OutputOperands
                      : InputOperands
                      : Clobbers
                      : GotoLabels)
```

关键词`asm`是GNU扩展的一个关键词，当代码能用`-ansi`和`-std`选项编译时，建议使用`__asm__`替代asm。

**asm-qualifiers**有三个可选项：

1. volatile
   - 典型场景是将input value转换为output value
   - 加上该关键字，严禁将此处的汇编语句与其它的语句重组合优化。例如，该段汇编代码的```output```为空时，若不加上volatile，则编译器可能会优化，但是加上就一定按照原来的语句进行执行。
2. inline
   - 使用该标记，会使得该段汇编代码在使用时size最小
3. goto
   - 代码跳转

## 汇编指令

在实现SpinLock时，涉及到的一些汇编指令如下所示，下面依次介绍具体语句的意思。

```c++
/* Compile read-write barrier */
#define barrier() asm volatile("": : :"memory")

/* Pause instruction to prevent excess processor bus usage */
#define cpu_relax() asm volatile("pause\n": : :"memory")

#define __HLE_ACQUIRE ".byte 0xf2 ; "
#define __HLE_RELEASE ".byte 0xf3 ; "

#define cpu_pause_v2()  __asm__ (".byte 0xf3, 0x90")
```

### Pause

Pause指令，会减少CPU的消耗，节省电量。指令的本质功能是：让加锁失败的CPU睡眠大约30个clock，从而使得读操作的频率低很多，流水线重排的代价也会很小。

**官方解释：**

> Description
> Improves the performance of spin-wait loops. When executing a “spin-wait loop,” a Pentium 4 or Intel Xeon processor suffers a severe performance penalty when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, it is recommended that a PAUSE instruction be placed in all spin-wait loops.
>
> An additional function of the PAUSE instruction is to reduce the power consumed by a Pentium 4 processor while executing a spin loop. The Pentium 4 processor can execute a spin-wait loop extremely quickly, causing the processor to consume a lot of power while it waits for the resource it is spinning on to become available. Inserting a pause instruction in a spin-wait loop greatly reduces the processor's power consumption.
>
> This instruction was introduced in the Pentium 4 processors, but is backward compatible with all IA-32 processors. In earlier IA-32 processors, the PAUSE instruction operates like a NOP instruction. The Pentium 4 and Intel Xeon processors implement the PAUSE instruction as a pre-defined delay. The delay is finite and can be zero for some processors. This instruction does not change the architectural state of the processor (that is, it performs essentially a delaying no-op operation).
>
> This instruction's operation is the same in non-64-bit modes and 64-bit mode.

**翻译过来就是：**

PAUSE指令提升了自旋等待循环（spin-wait loop）的性能。当执行一个循环等待时，Intel P4或Intel Xeon处理器会因为检测到一个可能的内存顺序违规（memory order violation）而在退出循环时使性能大幅下降。PAUSE指令给处理器提了个醒：这段代码序列是个循环等待。处理器利用这个提示可以避免在大多数情况下的内存顺序违规，这将大幅提升性能。因为这个原因，所以推荐在循环等待中使用PAUSE指令。

PAUSE的另一个功能就是**降低Intel P4在执行循环等待时的耗电量**。Intel P4处理器在循环等待时会执行得非常快，这将导致处理器消耗大量的电力，而在循环中插入一个PAUSE指令会大幅降低处理器的电力消耗。

**nginx代码示例**

```c++
// 以下代码引用自ngix源码的ngx_gcc_atomic_amd64.h文件
/* old "as" does not support "pause" opcode */
#define ngx_cpu_pause()         __asm__ (".byte 0xf3, 0x90")

// 以下代码引用自ngix源码的ngx_gcc_atomic_x86.h文件
#define ngx_cpu_pause()         __asm__ ("pause")
```

在`x86`系列架构中，`pause`指令也会被翻译成`0xf3, 0x90`, 因此，但是没有`pause`指令的机器会将其视为NOP指令。

[参考一](https://github.com/vaynedu/nginx-1.16.0/issues/48)

### memory

> The `"memory"` clobber tells the compiler that the assembly code performs memory reads or writes to items other than those listed in the input and output operands (for example, accessing the memory pointed to by one of the input parameters). To ensure memory contains correct values, GCC may need to flush specific register values to memory before executing the `asm`. Further, the compiler does not assume that any values read from memory before an `asm` remain unchanged after that `asm`; it reloads them as needed. Using the `"memory"` clobber effectively forms a read/write memory barrier for the compiler.
>
> Note that this clobber does not prevent the *processor* from doing speculative reads past the `asm` statement. To prevent that, you need processor-specific fence instructions.

**ngix代码示例**

```c++
// 以下代码引用自ngix源码的ngx_gcc_atomic_x86.h文件
/*
 * on x86 the write operations go in a program order, so we need only
 * to disable the gcc reorder optimizations
 */
#define ngx_memory_barrier()    __asm__ volatile ("" ::: "memory")

// 以下代码引用自ngix源码的ngx_gcc_atomic_amd64.h文件
#define ngx_memory_barrier()    __asm__ volatile ("" ::: "memory")
```

简单来说，就是告诉编译器内存的内容可能被更改了，需要无效所有Cache，并访问实际的内容，而不是Cache。

即memory强制gcc编译器假设RAM所有内存单元均被汇编指令修改，这样cpu中的registers和cache中已缓存的内存单元中的数据将作 废。cpu将不得不在需要的时候重新读取内存中的数据。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。

### .byte 0xf2;

- [`0xF2`] REPNE/REPNZ prefix *or* BND prefix

### .byte 0xf3;

- [`0xF3`] REP or REPE/REPZ prefix

## SpinLock实现

代码引用自[github spinlock](https://github.com/cyfdecyf/spinlock)

### spinlock-cmpxchg.h

```c
#ifndef _SPINLOCK_CMPXCHG_H
#define _SPINLOCK_CMPXCHG_H

typedef struct {
    volatile char lock;
} spinlock;

#define SPINLOCK_ATTR static __inline __attribute__((always_inline, no_instrument_function))

/* Pause instruction to prevent excess processor bus usage */
#define cpu_relax() asm volatile("pause\n": : :"memory")

SPINLOCK_ATTR char __testandset(spinlock *p)
{
    char readval = 0;

    asm volatile (
            "lock; cmpxchgb %b2, %0"
            : "+m" (p->lock), "+a" (readval)
            : "r" (1)
            : "cc");
    return readval;
}

SPINLOCK_ATTR void spin_lock(spinlock *lock)
{
    while (__testandset(lock)) {
        /* Should wait until lock is free before another try.
         * cmpxchg is write to cache, competing write for a sinlge cache line
         * would generate large amount of cache traffic. That's why this
         * implementation is not scalable compared to xchg based one. Otherwise,
         * they should have similar performance. */
        cpu_relax();
    }
}

SPINLOCK_ATTR void spin_unlock(spinlock *s)
{
    s->lock = 0;
}

#define SPINLOCK_INITIALIZER { 0 }

#endif /* _SPINLOCK_CMPXCHG_H */


```

### spinlock-xchg

```c

#ifndef _SPINLOCK_XCHG_H
#define _SPINLOCK_XCHG_H

/* Spin lock using xchg.
 * Copied from http://locklessinc.com/articles/locks/
 */

/* Compile read-write barrier */
#define barrier() asm volatile("": : :"memory")

/* Pause instruction to prevent excess processor bus usage */
#define cpu_relax() asm volatile("pause\n": : :"memory")

static inline unsigned short xchg_8(void *ptr, unsigned char x)
{
    __asm__ __volatile__("xchgb %0,%1"
                :"=r" (x)
                :"m" (*(volatile unsigned char *)ptr), "0" (x)
                :"memory");

    return x;
}

#define BUSY 1
typedef unsigned char spinlock;

#define SPINLOCK_INITIALIZER 0

static inline void spin_lock(spinlock *lock)
{
    while (1) {
        if (!xchg_8(lock, BUSY)) return;
    
        while (*lock) cpu_relax();
    }
}

static inline void spin_unlock(spinlock *lock)
{
    barrier();
    *lock = 0;
}

static inline int spin_trylock(spinlock *lock)
{
    return xchg_8(lock, BUSY);
}

#endif /* _SPINLOCK_XCHG_H */

```



### spinlock-xchg-backoff

```c

#ifndef _SPINLOCK_XCHG_BACKOFF_H
#define _SPINLOCK_XCHG_BACKOFF_H

/* Spin lock using xchg. Added backoff wait to avoid concurrent lock/unlock
 * operation.
 * Original code copied from http://locklessinc.com/articles/locks/
 */

/* Compile read-write barrier */
#define barrier() asm volatile("": : :"memory")

/* Pause instruction to prevent excess processor bus usage */
#define cpu_relax() asm volatile("pause\n": : :"memory")

static inline unsigned short xchg_8(void *ptr, unsigned char x)
{
    __asm__ __volatile__("xchgb %0,%1"
                :"=r" (x)
                :"m" (*(volatile unsigned char *)ptr), "0" (x)
                :"memory");

    return x;
}

#define BUSY 1
typedef unsigned char spinlock;

#define SPINLOCK_INITIALIZER 0

static inline void spin_lock(spinlock *lock)
{
    int wait = 1;
    while (1) {
        if (!xchg_8(lock, BUSY)) return;
    
        // wait here is important to performance.
        for (int i = 0; i < wait; i++) {
            cpu_relax();
        }
        while (*lock) {
            wait *= 2; // exponential backoff if can't get lock
            for (int i = 0; i < wait; i++) {
                cpu_relax();
            }
        }
    }
}

static inline void spin_unlock(spinlock *lock)
{
    barrier();
    *lock = 0;
}

static inline int spin_trylock(spinlock *lock)
{
    return xchg_8(lock, BUSY);
}

#endif /* _SPINLOCK_XCHG_BACKOFF_H */

```



### spinlock-xchg-hle

```c
#ifndef _SPINLOCK_XCHG_H
#define _SPINLOCK_XCHG_H

/* Spin lock using xchg.
 * Copied from http://locklessinc.com/articles/locks/
 */

/* Compile read-write barrier */
#define barrier() asm volatile("": : :"memory")

/* Pause instruction to prevent excess processor bus usage */
#define cpu_relax() asm volatile("pause\n": : :"memory")

#define __HLE_ACQUIRE ".byte 0xf2 ; "
#define __HLE_RELEASE ".byte 0xf3 ; "

static inline unsigned short xchg_8(void *ptr, unsigned char x)
{
    __asm__ __volatile__(__HLE_ACQUIRE "xchgb %0,%1"
                :"=r" (x)
                :"m" (*(volatile unsigned char *)ptr), "0" (x)
                :"memory");

    return x;
}

#define BUSY 1
typedef unsigned char spinlock;

#define SPINLOCK_INITIALIZER 0

static inline void spin_lock(spinlock *lock)
{
    while (1) {
        if (!xchg_8(lock, BUSY)) return;
    
        while (*lock) cpu_relax();
    }
}

static inline void spin_unlock(spinlock *lock)
{
    __asm__ __volatile__(__HLE_RELEASE "movb $0, %0"
            :"=m" (*lock)
            :
            :"memory");
}

static inline int spin_trylock(spinlock *lock)
{
    return xchg_8(lock, BUSY);
}

#endif /* _SPINLOCK_XCHG_H */


```

### 性能对比

cmpxchg < xchg < xchg-backoff < xchg-hle

**测试条件**

总叠加数为1000万，并发量（线程数）分别为1， 5， 10，每种场景测试三遍。

```shell
function run_test() {
    for nthr in 1 5 10; do
        ./$1 $nthr > /dev/null
        for i in `seq 1 3`; do
            ./$1 $nthr
        done
        echo
    done
}
```
**性能测试结果**
```tex
test spin lock using cmpxchg
0.104997	0.108640	0.110231	
1.502738	1.780505	1.588555	
3.188721	3.372979	3.288685	
test spin lock using xchg
0.102495	0.105192	0.101462	
1.511772	1.467292	1.315992	
1.966999	1.767013	1.607366	
test spin lock using xchg-backoff
0.102869	0.102168	0.107096	
0.210090	0.148667	0.115931	
0.691257	0.183631	0.194179	
test spin lock using xchg-hle
0.248885	0.248751	0.249399	
0.061266	0.056018	0.054837	
0.042291	0.052939	0.036410
```

