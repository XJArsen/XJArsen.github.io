---
title: 初探io_uring和liburingcopy
date: 2024-7-16 21:00 +0800
categories: [I/O]
tags: [linux]     # TAG names should always be lowercase

math: true
mermaid: true
---

> 最近有关注到io_uring，那就以它作为第一篇正式blog。

# 起因

先总结一下性能吧：根据互联网资料，主要优势在磁盘I/O方面。虽然在网络I/O方面，相较于基于epoll的非阻塞同步IO提升其实很小，但个人认为io_uring相对而言发展空间更大。计划待后续逐渐掌握了再进行个人的测试。



# io_uring

`io_uring` 是 2019 年 **Linux 5.1** 内核首次引入的高性能 **异步 I/O 框架**，能加速I/O密集型应用的性能。关于Linux的异步I/O此前用的是主要是AIO，在网络编程方面会使用ASIO。



## 核心数据结构

每个io_uring都有两个环形队列，在内核态和用户态共享。

* 提交队列（submission queue或 SQ）
* 完成队列（completion queue或 CQ）
* SQE： 描述需要完成的 I/O 操作并将其添加到提交队列 （SQ） 的尾部
* CQE：描述需要携带与操作结果相关的信息

有兴趣可以通过`man io_uring`自行查看。下面是简单介绍与说明：

使用方式：要发出的 I/O 请求放在 SQ，而内核将这些操作的结果放在CQ。

两者都为单生产者、单消费者模式。在提交I/O时，应用程序是生产者，内核是消费者。在完成I/O时相反，应用程序是消费者，内核是生产者。为了防止CQ溢出，默认情况下CQ的大小是SQ的两倍。

具体的数据索引方式比较负载暂不做探讨

## 高效原理

高效主要在以下几个大方面：

* io_uring采用共享缓冲区，即使用`mmap`减少模式开销

* 内部使用内存排序，通过read_barrier()、write_barrier()的同步方式，对外提供无锁接口。
* 支持轮询模式，相比中断模式省去了系统调用，但是更销毁cpu资源

由于目前目标还在掌握使用阶段，先不做细致研究，



# liburing

`liburing`是io_uring的用户态库，便于io_uring的使用。

> 仓库地址：[axboe/liburing at liburing-2.6 (github.com)](https://github.com/axboe/liburing/tree/liburing-2.6)
> API手册：[Manpages of liburing-dev in Debian unstable — Debian Manpages](https://manpages.debian.org/unstable/liburing-dev/index.html)

## 安装与编译

```shell
# 拉取仓库
git clone git@github.com:axboe/liburing.git

# 切换版本
cd liburing
git checkout -b liburing-2.6 tags/liburing-2.6

# 运行configure
./configure

# 编译安装
make
sudo make install

# 检查安装
find / -name "liburing.so*" 2>/dev/null
```



## 数据结构

struct io_uring

```c
struct io_uring {
	struct io_uring_sq sq;	// 提交队列
	struct io_uring_cq cq;	// 完成队列
	unsigned flags;			// poll模式下开启的内核线程绑定的cpu
	int ring_fd;			// 该实例文件描述符
	 // 内核当前支持的能力，内核设置
	unsigned features;
	int enter_ring_fd;
	__u8 int_flags;
	__u8 pad[3];
	unsigned pad2;
};
```

## API

### io_uring_queue_init

```c
// 初始化一个io_uring实例
// @param : entries:队列长度 ring:io_uring实例 	flags:配置,0表示默认配置
// @return : 成功返回0,失败时返回 -1 并设置 errno
int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);						
// example
struct io_uring ring;
ret = io_uring_queue_init(QD, &ring, 0);    
...
// 销毁
io_uring_queue_exit(&ring);
```



### io_uring_register_files

```c
// 注册用于异步 I/O 的文件或用户缓冲区
// @param : fd:io_uring 描述符 opcode:操作码 arg:操作指定的参数 nr_args:参数数量
// @return : 成功时返回 0，失败时返回-1并设置 errno
int io_uring_register(unsigned int fd, unsigned int opcode, void *arg, unsigned int nr_args);
```



### io_uring_get_sqe

```c
// 获取一个sqe,一般配合io_uring_prep_readv使用
// @param : ring:io_uring实例
// @return : 返回可用的sqe
struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);

```



### io_uring_prep_($option)

作用：将 sqe 提交到提交队列中
相关的操作种类众多：read、write、send等

通过`io_uring_sqe_set_flags`指定标志来更改提交队列条目的行为

`io_uring_sqe_set_data` 传入data。 操作完成后通过 `io_uring_cqe_get_data` 从 cqe 中读出data

*  io_uring_sqe_set_data

```c
// 存储一个带有提交队列条目 sqe 的user_data指针
void io_uring_sqe_set_data(struct io_uring_sqe *sqe, void *user_data);
void io_uring_sqe_set_data64(struct io_uring_sqe *sqe, __u64 data);
```

* io_uring_cqe_get_data

```c
// user_data 使用完成队列条目 CQE 作为数据指针
void *io_uring_cqe_get_data(struct io_uring_cqe *cqe);
__u64 io_uring_cqe_get_data64(struct io_uring_cqe *cqe);
```



### io_uring_submit

```c
// 通知 io_uring 从提交队列中消费 sqe
// @return: 返回提交成功的 sqe 数量
int io_uring_submit_and_get_events(struct io_uring *ring);
```



### io_uring_wait_cqe

```c
// 阻塞直到有 cqe 返回
// @return: 成功时返回 0，失败时返回-1并设置 errno
int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);
```



### io_uring_peek_cqe

```c
// 非阻塞 如果没有就绪的 cqe，则直接报错返回
int io_uring_peek_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);
```



### io_uring_cqe_seen

```c
//  更新io_uring 实例的完成队列
void io_uring_cqe_seen(struct io_uring *ring, struct io_uring_cqe *cqe);
```



简单地了解API后，就能开始阅读代码了

作者给的示例：[liburing/examples at liburing-2.6 · axboe/liburing (github.com)](https://github.com/axboe/liburing/tree/liburing-2.6/examples)

unixism：[Welcome to Lord of the io_uring — Lord of the io_uring documentation (unixism.net)](https://unixism.net/loti/)

# Reference

[[译\] Linux 异步 I/O 框架 io_uring：基本原理、程序示例与性能压测（2020） (arthurchiao.art)](https://arthurchiao.art/blog/intro-to-io-uring-zh/)

[io_uring.pdf (kernel.dk)](https://kernel.dk/io_uring.pdf)
