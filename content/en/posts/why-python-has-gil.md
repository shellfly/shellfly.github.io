---
title: "Why Python has a GIL"
date: 2018-03-19T17:14:00+08:00

---

## GIL
GIL is an acronym of Global Interpreter Lock, which is used by CPython to guarantee that there is only one thread executes bytecode at once.
Because the exist of GIL, multi threads program would execute serially rather then concurrent even if the OS schedules different threads to different CPU cores. Therefore, the efficiency of multi threads program will reduce significantly.
So, why does Python have a GIL?
There is a short explain in Python official document:
> Python interpreter is not fully thread-safe. In order to support multi-threaded Python programs, there’s a global lock, called the global interpreter lock or GIL.

## Thread Safe
If you write a multi threads program using C or Java, you have to use lock to protect share resources between threads. CPython is a interpreter implemented by C, it also need lock mechanism to keep code right, and GIL is a super big lock. Whenever a thread starts to execute, it has to acquire the GIL, and other threads will remain waiting while there is a running thread. Other system like JVM uses fine-grained lock to guarantee thread safe.

There are two common operations will lead to the release of GIL. One is cooperative, a thread will release GIL before it doing I/O operations. Another one is preemptive, a thread has to release a GIL after a certain period of time.
GIL in python doesn’t mean you don’t have to care about thread safe when you are writing Python code. The GIL protects the Python interpreter internals, but if you have some logic depend on serial code execution, you still have to add a lock to guarantee the right of you code.

## Why GIL
The most important reason for having a GIL is that it makes a single thread program very fast. There are no extra acquiring or releasing operations compared with fine-grained locks.
It also makes it easy to integrate a C library. If a library is not thread safe, you just have to acquire GIL before calling it.

PS:
The human brain also has a GIL like a Python interpreter, it can only process one thing at a time. So, don’t watch your phone while driving a car, you should remember this after learning some Python.