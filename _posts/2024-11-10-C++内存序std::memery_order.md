---
title: C++内存序std::memery_order
date: 2024-11-10 20:00 +0800
categories: [c++]
tags: [c++]     # TAG names should always be lowercase

math: true
mermaid: true
---

> 在研究原子操作时，发现了此前未曾注意的内存访问顺序问题，简单做个笔记

# 简介
标准库的内存序主要是`std::memory_order`，在` <atomic>` 中定义，用于控制多线程环境中原子操作的顺序。
因为指令执行顺序会受编译器优化、缓冲、CPU的影响，所以程序在最终执行时并不会按照你之前的原始代码顺序来执行。因此出现了内存序，程序员、编译器、CPU之间的一套约定。

参考[std::memory_order - cppreference.com](https://zh.cppreference.com/w/cpp/atomic/memory_order)，以下摘录的定义和作用，

```c++
enum class memory_order : /* 未指明 */
{
    relaxed, consume, acquire, release, acq_rel, seq_cst
};
inline constexpr memory_order memory_order_relaxed = memory_order::relaxed;
inline constexpr memory_order memory_order_consume = memory_order::consume;
inline constexpr memory_order memory_order_acquire = memory_order::acquire;
inline constexpr memory_order memory_order_release = memory_order::release;
inline constexpr memory_order memory_order_acq_rel = memory_order::acq_rel;
inline constexpr memory_order memory_order_seq_cst = memory_order::seq_cst;
```

| memory_order | 作用 |
| ---------------------- | ------------------------------------------------------------ |
| memory_order_relaxed | 宽松操作：没有同步或定序约束，仅对此操作要求原子性 |
| memory_order_consume | 有此内存定序的加载操作，在其影响的内存位置进行*消费操作*：当前线程中依赖于当前加载的值的读或写不能被重排到此加载之前。其他线程中对有数据依赖的变量进行的释放同一原子变量的写入，能为当前线程所见。在大多数平台上，这只影响到编译器优化 |
| memory_order_acquire | 有此内存定序的加载操作，在其影响的内存位置进行*获得操作*：当前线程中读或写不能被重排到此加载之前。其他线程的所有释放同一原子变量的写入，能为当前线程所见。 |
| memory_order_release | 有此内存定序的存储操作进行*释放操作*：当前线程中的读或写不能被重排到此存储之后。当前线程的所有写入，可见于获得该同一原子变量的其他线程，并且对该原子变量的带依赖写入变得对于其他消费同一原子对象的线程可见 |
| memory_order_acq_rel | 带此内存定序的读修改写操作既是*获得操作*又是*释放操作*。当前线程的读或写内存不能被重排到此存储之前或之后。所有释放同一原子变量的线程的写入可见于修改之前，而且修改可见于其他获得同一原子变量的线程。 |
| memory_order_seq_cst | 有此内存定序的加载操作进行*获得操作*，存储操作进行*释放操作*，而读修改写操作进行*获得操作*和*释放操作*，再加上存在一个单独全序，其中所有线程以同一顺序观测到所有修改。 |

# 简单使用
官方的解释比较抽象，采用内存屏障的方式解释。
内存屏障简答来说就是在屏障前的操作必须生效后才能进行后续操作。
## 弱内存序
在之前先简单介绍一下弱内存序，这是个比较麻烦的问题
弱内存序问题产生的根本原因是两种架构的CPU具有不同的内存模型（x86：Total Store Order，Arm、RISC-V：Weak Memory Order）。x86的内存序问题要比Arm简单一些，x86架构下write memory操作写入内存必须经过write buffer，这是一个FIFO结构，可以严格保证顺序；只有read memory操作可以直接从write buffer或内存中读取，因此可能乱序到write memory之前。而Arm架构下不存在这样的数据结构保证顺序，所有的write memory和read memory操作都可能互相被重排。

## seq_cst 顺序一致性
这是原子变量默认的内存序。
能保证原子变量M之前的指令执行完毕，其原理是在语句后插入内存屏障保证有序，所以只有指令生效后才能执行原子指令。

```c++
// 定义
std::string data = "";
std::atomic_int flag = 0;

void func() {
    // A一定限于B执行
    data = "data"; // A
    // barrier
    flag.store(1); // B
    // barrier
}
```

## relaxed
relaxed比较简单，可以理解为只保证原子性，不保证内存序。
在x86上，因为强内存序，seq_cst和relaxed性能差异不大。一般用于Arm、RISC-V下等弱内存序中。

## release acquire
release用于保护自己，而acquire用于保护其他操作，只有一个屏障，无法发挥内存序的作用。acquire、release配合使用。写入用acquire, 读取用release.
```c++
flag.store(1, std::memory_order_release);
flag.load(1, std::memory_order_acquire);
```
acq_rel是两者的结合，用于既有读取又有写入的情况，如fetch_add()，汇编角度看fetch_add如下：
```assembly
r0 = load;
add r0, r0, #1;
store r0;
```

以fetch_add()为例，在内存屏障角度看这三者：
```c++
// barrier
count.fetch_add(1, std::memory_order_release);


count.fetch_add(1, std::memory_order_acquire);
// barrier


// barrier
count.fetch_add(1, std::memory_order_acq_rel);
// barrier
```
在汇编角度看
```c++
count.fetch_add(1, std::memory_order_release);
// r0 = load; // relaxed
// add r0, r0, #1;
// store r0; // release

count.fetch_add(1, std::memory_order_acquire);
// r0 = load; // acquire
// add r0, r0, #1;
// store r0; // relaxed

count.fetch_add(1, std::memory_order_acq_rel);
// r0 = load; // acquire
// add r0, r0, #1;
// store r0; // release
```
## 对比seq_cst和acq_rel
简单来说seq_cst比acq_rel更加严格，表现在
* seq_cst 可以对任何操作进行指定。acq_rel 只是对 rmw 操作进行特化的 flag。使用范围就不一样。
* seq_cst 可以保证全序，在任何线程上顺序都是一致的

# Reference
https://en.cppreference.com/w/cpp/atomic/memory_order
https://www.bilibili.com/video/BV1Qy411q7Xq/
https://www.zhihu.com/question/561700714
