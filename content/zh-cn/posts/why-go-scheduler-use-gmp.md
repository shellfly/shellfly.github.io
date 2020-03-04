---
title: "为什么Go的调度模型是GMP，而不是GM"
date: 2020-03-04T20:13:27+08:00
draft: false
---

最喜欢Go里面的一个功能就是goroutine，它提供了一个简单但是强大的语言级别的并发机制。虽然更喜欢写Python代码，不过说到并发，goroutine比Python里的yield和asyncio都好理解的多，而且性能理论上也好很多。

为了管理goroutines，Go有自己的调度器。你可能已经知道Go的调度器使用G-M-P模型来调度goroutine。最近读一些关于Go的调度器怎么工作的文章时，一个问题开始浮现在脑子里面，为什么调度器需要一个P呢？为什么不直接就把goroutine（G）和系统的thread（M）做映射，那样不是更简单直接吗？

继续阅读更多的文章后，我发现，在最开始Go 1.0发布的时候，调度器里确实是没有P的，就是用thread直接来调度goroutine，但是有一些性能上的问题，所以在Go 1.1的时候就引入了一种新的调度器，也就是从那时候起，调度器开始使用GMP模型。这是最开始的[设计文档](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw).

文档里提到了在没有P的时候，调度器里的4个问题

1. 单一全局的mutex和state
2. goroutine在不同的thread之间传递的开销
3. 大量无用的M的内存缓存
4. 因为系统调用产生的阻塞和解除阻塞

调度器需要用一个全局的mutex来管理和维护所有goroutine的调度状态，很难扩展。goroutine经常需要在不同的thread间传递，增加的额外的延时和开支。只有当前在运行的thread才需要内存缓存以及其他一些像堆栈的资源，被系统调用阻塞的thread其实并不需要。

为了解决上面这些问题，Go运行时里引入了P（处理器），所以现在就有了M代表系统的thread，P代表执行Go代码需要的资源。P在M和G中间，它会获取下一个可执行的goroutine并且找到一个M来运行它。被系统调用阻塞的thread会释放P，这样P可以继续使用其他M或者创建一个新的M来处理其他goroutine。

引入P后，调度器就有了分散的执行队列，用来执行goroutine的资源被限制在有限的P上面，因为系统调用产生的阻塞和解除阻塞也变的更高效。


ref:

https://taohuawu.club/high-performance-implementation-of-goroutine-pool

https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/

http://morsmachine.dk/go-scheduler

https://github.com/golang/go/blob/master/src/runtime/proc.go



