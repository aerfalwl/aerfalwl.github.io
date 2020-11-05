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

acquire于内存的load操作对应，也就是读操作。设置该标志，表示在这条语句之后设置了barrier。

即所有后续操作的读操作必须在本条原子操作完成之后执行。

### memory_order_release

release 对应于内存的 store操作，也就是写内存。设置标志，表示在这条语句之前设置了barrier。即所有之前的写操作完成之后才能执行本条原子操作。

该语句和memory_order_qcquire有点相似，但是如何`见字如面`呢，即如何第一眼看到便区分开来呢。我是这样区分的：

1. acquire和release的对象都是barrier；
2. acquire即请求，请求之后，便得到了barrier，因此是在这条语句之后设置了barrier，因此后续的读(load)操作必须在本条原子操作完成之后；
3. release即释放，要想释放，必须先拥有，即该条语句之前已经拥有了barrier，因此之前的写(store)操作完成之后，才能执行本条原子操作；
4. acquire操作常用来读取数据的同步，release操作常用来写数据的同步。

### memory_order_acq_rel

对读取和写入施加acquire-release语句，无法被重排。

可以看见其他线程施加release语义的所有写入，同时自己的release结束后所有写入对其施加acquire语义的线程可见。

### memory_order_seq_cst

