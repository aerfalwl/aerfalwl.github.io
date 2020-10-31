---
title: C++六种内存模型
date: 2020-10-30 11:29:45
categories: Memory
tags:
	- C++
	- Memory 
---

## C++内存模型

C++规定了6种访存顺序

```C++
enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
};

```

这6种内存模型可以分为3类

1. 顺序一致性模型, memory_order_seq_cst
2. Acquire-release模型，也称为获取释放内存模型。
   - memory_order_consume
   - memory_order_acquire
   - memory_order_release
   - memory_order_acq_rel
3. Relax模型，即memory_order_relaxed，宽松的内存序列化模型

### memory_order_relaxed

只保证当前操作的原子性，不考虑线程之间的同步，其它线程可以读到新值，也可以读到旧值。

### memory_order_consume



### memory_order_acquire

可以理解为mutex的lock操作。





### memory_order_release

可以理解为mutex的unlock操作。



### memory_order_acq_rel

对读取和写入施加acquire-release语句，无法被重排。

可以看见其他线程施加release语义的所有写入，同时自己的release结束后所有写入对其施加acquire语义的线程可见。

### memory_order_seq_cst

