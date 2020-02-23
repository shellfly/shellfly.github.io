---
title: "为什么Python有一个GIL"
date: 2018-03-19T17:14:00+08:00

---

GIL
===

GIL是Global Interpreter Lock的缩写，也就是全局的解释器锁。CPython用GIL来保证同一时间只有一个线程在执行Python代码。所以即使多个线程被操作系统分配到CPU不同的核里，最终的效果还是线性执行，没法利用多核的优势。这也就导致了多线程程序的执行效率大大降低。

所以Python里到底为什么一定要有一个GIL呢？

Python官方文档中关于GIL有一段简要的说明：

> Python interpreter is not fully thread-safe. In order to support multi-threaded Python programs, there’s a global lock, called the global interpreter lock or GIL.

简单翻译一下就是：因为Python解释器（CPython）自身不是完全线程安全的，所以为了支持多线程的Python程序，需要有一个全局解释器锁，也就是GIL。

线程安全（Thread Safe）
======================

如果用C或者Java写过多线程程序，就会知道访问共享资源的时候都是需要加锁的。CPython顾名思义，就是用C实现的Python解释器，当启用多线程的时候，为了保证代码的正确性，就需要锁，而GIL就是一把超级大锁，每个线程开始执行之前都需要先获得GIL，其他没有获得GIL的线程保持在等待状态，而像JVM中则采用细粒度(fine-grained)的锁来保证线程安全。

通常有两种操作会导致线程释放GIL，一种是协作式的（cooperative），在执行I/O操作之前，线程会主动释放GIL。另一种是抢占式的（preemptive），线程执行一定时间后会被迫释放GIL。

但是并不是有了GIL后，就不需要再关心线程安全了。GIL只能保证Python的内部数据结构同一时间只有一个线程可以访问，其实是保证了单独的操作是线程安全的，不需要额外的保护措施。但是如果有特定顺序的连续调用，还是需要用户显式的加锁来控制程序的正确性，也就是《深入理解Java虚拟机》中描述的相对线程安全。

GIL的优点
=========

使用GIL最重要的原因就是：它可以让单线程程序的速度非常快，不需要考虑锁的释放和获取，和用细粒度的锁来保证解释器状态相比，省了很多不必要的操作。

在集成C库时，也不需要担心线程安全，如果不是线程安全的，只需要在调用的时候先获取GIL即可。


PS:
人脑和python解释器一样也有个GIL，一次只能处理一件事，通过context switch来模拟多线程运行。所以学过Python后一定要记得一件事，开车时不要看手机。
