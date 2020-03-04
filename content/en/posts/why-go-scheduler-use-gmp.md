---
title: "Why Go scheduler uses GMP"
date: 2020-03-04T20:13:27+08:00
draft: false
---

The most beautiful feature I like in Go is goroutine, which is a simple and powerful language level concurrency mechanism. I prefer to write Python code than Go. But speak of concurrency, goroutine is more intuitive and easy to understand than `yield` and `asyncio` in Python. And performance is better in theory.

To manage goroutines, Go has it's own scheduler and you propably already know there is a GMP model which Go sheduler is using. When I read articles about how Go scheduler works, A question come to my mind that why there is a `P`? and why not just map goroutines(G) to OS threads(M) directly?

After reading more documents, I find at beginning when Go just release 1.0, there is no `P` but with a few performances issues in the scheduler. And in Go 1.1 a new scheduler was added and since then the GMP model is used in the scheduler. Here is a [original design doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw).

There are four issues mentioned in the document about the scheduler without `P`

1. Single global mutex and centralized state
2. Goroutine hand-off
3. Per-M memory cahce
4. Aggresive thread blocking/unblocking

The scheduler has to use mutex to maintain a centralized state of all goroutines which inhibits scalability. Goroutines are hand-off in different threads which increased latencies and additional overheads.
Only running threads(M) need cache, when a thread(M) is blocked in system call, it doesn't need so much resources.
Because of the system call a thread can be blocked and unblocked very often, this adds a lot of overhead to schedule goroutines.

To solve the issues above. A notion of P(Processors) was introduced into runtime. so now we have M which represents OS thread, P represents a resource to execute Go code. P is in the middle of G and M. P will continue get next runable G in its run queue and run it in M. and when M is idle or in syscall, it doesn't need a P.

By introducing P, the shceduler can have decentralize run queue which is bind to every P, the resources for running a G is only exist in P and blocked/unblocked threads can be handled more efficiency.


Ref:

http://morsmachine.dk/go-scheduler

https://github.com/golang/go/blob/master/src/runtime/proc.go



