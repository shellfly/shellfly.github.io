---
title: "互斥锁和原子操作（Mutex and Atomic）"
date: 2021-10-12T11:02:38+08:00
draft: false
tags: ["computer science"]
---

## 信号量的发明和问题

Edsger W. Dijkstra[^1] 在1965年左右研究并发编程的时候第一次提出了互斥（mutual exclusion）的问题，并且提出了最早的一种同步原语（synchronization primitives）：信号量（semaphore）。 Dijkstra是丹麦人，他用两个丹麦的单词首字母`P`和`V`来作为信号量的两种基本的操作, 可以把`P`理解成`down`或者`wait`, `V`表示`up`或者`signal`, 只需要在访问临界区前后使用对应的操作，就能让并发的程序安全的访问临界区：

![critical section](/images/critical-section.png)

信号量作为最早提出的一种同步原语，设计的非常简单，可以通过把信号量`S`的初始值设置成不同的值来实现不同的效果：把`S`的初始值设置成一个大于1的值，就可以用来给资源计数；把`S`的初始值设置成1，就能实现类似互斥锁的效果；把初始值设置成0就能用来等待通知。

这种灵活的使用方法也带来了很多潜在的问题，比如代码里忘记调用`P`而直接调用了`V`,导致错误的释放，或者递归的调用`P`导致的死锁，或者任务中断没有释放导致的死锁等等。因为存在着这些问题，信号量的使用需要格外的注意。而且因为功能太多，也不利于直观的理解代码到底是在做什么，所以现在的编程语言或者系统中一般都会提供一些更专用的同步原语，其中最常见的就是互斥锁。

## 互斥锁

互斥锁一般用`mutex`表示，它是MUTual EXclusion的首字母组合出来的词，日常并发编程的时候使用的`锁`, 像是Python中的`threading.Lock` 和 Golang里的 `sync.Mutex`就是一种互斥锁。
互斥锁的作用有点像是初始值是1的信号量，不过在一些实现中，互斥锁和信号量还有一个重要的区别是互斥锁的释放只能由获取到互斥锁的线程来完成，然而这并不是一个通用的设计，在Python[^2]和Go[^3]里，锁的释放都是是允许在其他线程或者goroutine里来进行的，原因可能是这种实现比较简单， 实际的代码虽然很少这么用, 这个Go issue[^4]里有一些是不是要禁用这种方式的讨论。

## 互斥锁的实现

锁的存在是为了解决编发编程中一个常见的问题：保证一段代码的原子性。以一个简单的加一操作为例,加锁的代码看起来一般是这样：

```
var mutex
lock(mutex)
balance = balance + 1
unlock(mutex)
```

如果在一个单CPU的机器上，只要通过禁用和启用中断，就能达到`lock`和`unlock`的效果，这个方法最简单，但是并不怎么实用，首先中断需要很高的权限才能控制，其次在多CPU的情况下无法起作用。所以我们需要更可用的方式，一个简单的想法是通过一个`flag`变量来实现：

```
# mutex.flag = 0

lock(mutex) {
    while mutex.flag == 1 {
        // spin-wait
    }
    // set the flag, acquire the lock
    mutex.flag = 1
}

unlock(mutex) {
    mutex.flag = 0
}
```

看起来好像可以，但是要想到`lock`函数本身的代码也是会有并发问题的，如果要想实现一个可用的`lock`函数，则又需要一个机制来保证`lock`函数里那几行代码本身的原子性，这似乎是一个先有鸡还是先有蛋的问题，不过CPU的设计已经考虑到了这一点，从硬件级别提供了一些原子操作可以用来实现锁。

## 原子操作

高级语言的代码通过编译或者解释最终都变成一条条CPU指令在CPU上运行，一条CPU指令是CPU上运行的最小单位，也就是一个不可分割的原子操作，如果代码被编译成多条CPU指令，因为并发的存在，就不能保证代码执行的原子性。`lock`里面的代码做了两件事，一个是比较`flag`是不是等于特定的值，一个是给`flag`赋值，如果CPU提供了一条指令来同时做这两件事，那就可以用它来实现锁。常见的用来实现锁的指令有Test-and-set[^5], Compare-and-swap[^6]等。用Test-and-set举例：

```
lock(mutex) {
    while (TestAndSet(lock.flag, 1) == 1){    
        // spin-wait
    }
}
```

硬件提供的原子操作是正确实现锁的理论基础，不过锁的实现除了正确性以外还需要考虑另外两个重要的因素：公平和性能，Linux系统里的futex[^7]通过尽可能减少了系统调用和避免无用的上下文切换，提供了一个很好的机制来实现锁。

## 更多的原子操作

既然有了锁，在多线程修改共享变量的时候，只要加个锁，问题就都能解决了，但是这是不是最好的方法呢？以上面的`balance = balance + 1`为例，我们已经知道锁的底层实现其实是通过硬件提供的原子操作，而像加1这种操作本来就有对应的原子操作，那我们应该是用锁还是原子操作呢？

性能是其中一个要考虑的因素，锁在原子操作基础之上加了很多其他控制用来实现锁的获取和释放，当锁已经被另外一个线程获取时甚至需要系统调用、挂起当前线程等操作，所以通常来说原子操作本身的性能比使用锁要好。另外，锁本身也会带来一些问题，比如死锁，比如实时系统并不想阻塞一个线程等等，非阻塞算法[^8]就是对这些问题专门的研究，尽量通过原子操作来实现无锁的并发编程。

不管怎样，互斥锁还是一个并发编程中非常简单实用的工具，虽然有一些操作可以通过无锁的方式来实现，但是还是有很多限制。理解原理后，具体的问题就可以具体分析，选择最适合的工具去解决问题。

[^1]: https://en.wikipedia.org/wiki/Edsger_W._Dijkstra
[^2]: https://docs.python.org/3/library/threading.html#threading.Lock.release
[^3]: https://github.com/golang/go/blob/go1.17/src/sync/mutex.go#L173-L179
[^4]: https://github.com/golang/go/issues/9201
[^5]: https://en.wikipedia.org/wiki/Test-and-set
[^6]: https://en.wikipedia.org/wiki/Compare-and-swap
[^7]: https://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf
[^8]: https://en.wikipedia.org/wiki/Non-blocking_algorithm


