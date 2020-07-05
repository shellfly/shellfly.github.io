---
title: "Nonblocking IO (or how goroutine works)"
date: 2020-07-01T21:00:25+08:00
draft: false
---

There are two common patterns to design concurrency programs, Thread or Event-driven. They can both handle large concurrency requests but have different tradeoffs. Using thread means you have to consider synchronization, resource utilization, etc. Using event-driven means you can't utilize multiple core CPU.

The most frequent operations that concurrency programming handles is I/O. A program that handles large concurrency must be an I/O intensive program instead of a CPU intensive program. And I/O has two modes: Blocking and Nonblocking.

As a programming language designed for concurrency, **does Go use blocking or nonblocking I/O?**

To understand this question, we have to first understand the meaning of blocking and nonblocking. an I/O operation depends on external status(like network), when calling an I/O in blocking mode, it can blocks the program forever, while for nonblocking, it will return an error code to indicate the operation can't be complete now. The advantage of nonblocking is that it can be used together with multiplexing functions(epoll, kqueue), to achieve high concurrency effect.

All functions provided by Go is blocking, but this blocking is not blocking system thread but blocking the goroutine which runs the I/O operation. Underneath, Go uses nonblocking I/O that OS provides, this is a feature of the language to convert blocking to nonblocking, to make the concurrency code read and write straightforward.

The part that converts blocking I/O to nonblocking I/O is called [netpoller](https://morsmachine.dk/netpoller). whenever a goroutine read or write a file descriptor and receive an error code, it will call netpoller, tell it to notify the goroutine when it's ready to perform I/O again. netpoller was designed for network I/O, and later [support for file I/O was also added]((https://github.com/golang/go/commit/c05b06a12d005f50e4776095a60d6bd9c2c91fac)).

Does this mean all I/O in Go is nonblocking? the answer is: **No, because the most common file I/O, disk I/O doesn't have the concept of nonblocking.**

This is a limitation of OS itself, in Unix, the kernel provides system call to provide services, and this includes disk or network I/O. System calls can be divided by functions, some are for I/O operations, some are for memory management, some are for process scheduling. But it can also be divided by "slow" system calls and others, the "slow" system calls are that may block the program forever.

In *Advanced Programming in the UNIX Environment*, the author mentioned, these "slow" system calls include:

* Reads that can block the caller forever if data isn't present with certain file types (pipes, terminal devices, and network devices)
* Writes that can block the caller forever if the data can't be accepted immediately by these same file types (no room in the pipe, network flow control, etc.)
* Opens that block until some condition occurs on certain file types (such as an open of a terminal device that waits until an attached modem answers the phone, or an open of a FIFO for writing-only when no other process has the FIFO open for reading)
* Reads and writes of files that have mandatory record locking enabled
* Certain ioctl operations
* Some of the interprocess communication functions

And the book also mentioned that disk I/O are not considered slow, even though the read or write of a disk file can block the caller temporarily.

When you add a normal file descriptor to `select`, it will always report it as readable and writable. and `epoll` will return an error, forbid you to add a normal file descriptor.

So, in Go, when doing disk I/O, it not only parks the goroutine, it will operate as a normal system call, park the system thread to wait for the call to complete. Of course, this is not seen by Go users. All I/O operations are blocking from the language perspective, that's why it is designed for concurrency, you don't need to care about thread creation or destroy, either care about callback of event loop. Through goroutine, one can write code that easier to understand and achieve a good concurrency effect.


Disk I/O doesn't support nonblocking, but it can be asynchronous through AIO or the new Linux kernel technique [io_uring](https://lwn.net/Articles/810414/). Go community also started the discussion about how to [support new linux io_uring interface](https://github.com/golang/go/issues/31908). Thanks to the new kernel technology, we should be able to see better performance improvements on disk reads and writes in the near future.