---
layout: post
title: Goroutine 浅析
meta: Goroutine 浅析
draft: false
category: [goroutine, concurrency]
---

以下为 goroutine 学习过程中，搜索到的一些资料和一些思考，记录一下。

## 并发还是并行
> Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.[1]

* 并发是在同一时间处理（dealing with）多件事情。
* 并行是在同一时间做（doing）多件事情。

并发的目的在于把当个 CPU 的利用率使用到最高。并行则需要多核 CPU 的支持。

## CSP
Communicating Sequential Processes，译为通信顺序进程。是一种形式语言，用来描述并发系统间进行交互的模式。
例如：

> `COPY = *[c:character; west?c → east!c]`  
> the process repeatedly receives a character from the process named west, and then sends that character to process named east. The parallel composition  
> `[west::DISASSEMBLE || X::COPY || east::ASSEMBLE]`  
> assigns the names west to the DISASSEMBLE process, X to the COPY process, and east to the ASSEMBLE process, and executes these three processes concurrently.[3]  

CSP 通过把输入/输出和并发环境下的进程通信作为基础的方法和结构。定义了一套自己的原语，包括：并发执行，输入/输出，循环执行，条件判断等。然后用定义好的语法去实现一些常见的问题，比如：协程，数论，经典的同步问题（生产者消费者，哲学家就餐）。

通过 CSP 的定义，使并发编程能在更高的层次实现，编写的程序不需要关心底层的资源共享、加锁、调度切换等细节，使并发程序的编写更简单。[6]

Go 就是基于 CSP 的思想来实现的并发模型。

## Go runtime scheduler

### Why need Go scheduler?
主要有两个原因：

* 线程较多时，开销较大。
* OS 的调度，程序不可控。而 Go GC 需要停止所有的线程，使内存达到一致状态。[7]

### Struct
M 代表系统线程，G 代表 goroutine，P 代表 context。

### M:N
> There are 3 usual models for threading. One is N:1 where several userspace threads are run on one OS thread. This has the advantage of being very quick to context switch but cannot take advantage of multi-core systems. Another is 1:1 where one thread of execution matches one OS thread. It takes advantage of all of the cores on the machine, but context switching is slow because it has to trap through the OS.[7]

M:N 则综合两种方式（N:1, 1:1）的优势。多个 goroutines 可以在多个 OS threads 上处理。既能快速切换上下文，也能利用多核的优势。

### Context switch
在程序中任何对系统 API 的调用，都会被 runtime 层拦截来方便调度。
Goroutine 在 system call 和 channel call 时都可能发生阻塞[8]，但这两种阻塞发生后，处理方式又不一样的。

* 当程序发生 system call，M 会发生阻塞，同时唤起（或创建）一个新的 M 继续执行其他的 G。

> If the Go code requires the M to block, for instance by invoking a system call, then another M will be woken up from the global queue of idle M’s. This is done to ensure that goroutines, still capable of running, are not blocked from running by the lack of an available M.[11]

* 当程序发起一个 channel call，程序可能会阻塞，但不会阻塞 M，G 的状态会设置为 waiting，M 继续执行其他的 G。当 G 的调用完成，会有一个可用的 M 继续执行它。

> If a goroutine makes a channel call, it may need to block, but there is no reason that the M running that G should be forced to block as well. In a case such as this, the G’s status is set to waiting and the M that was previously running it continues running other G’s until the channel communication is complete. At that point the G’s status is set back to runnable and will be run as soon as there is an M capable of running it.[11]

### P 的作用：
* 每个 P 都有一个队列，用来存正在执行的 G。避免 Global Sched Lock。
* 每个 M 运行都需要一个 MCache 结构。M Pool 中通常有较多 M，但执行的只有几个，为每个池子中的每个 M 分配一个 MCache 则会形成不必要的浪费，通过把 cache 从 M 移到 P，每个运行的 M 都有关联的 P，这样只有运行的 M 才有自己的 MCache。[11]

## Goroutine vs OS thread
其实 goroutine 用到的就是线程池的技术，当 goroutine 需要执行时，会从 thread pool 中选出一个可用的 M 或者新建一个 M。而 thread pool 中如何选取线程，扩建线程，回收线程，Go Scheduler 进行了封装，对程序透明，只管调用就行，从而简化了 thread pool 的使用。[12]

## Goroutine vs Python yield
* 创建成本：Go 原生支持协程，通过 `go func()` 就可以创建一个 goroutine。Python 可以通过 `gevent.spawn` 来新建一个 coroutine，需要第三方库来支持。
* Goroutine 之间的通信更简单，通过 channel call 即可实现，上下文切换透明（只有少部分需要自己注意 Gosched）。Python 需要 yield 来传递数据和切换上下文（通过一些库封装后对调用者来说也是透明的，比如：gevent/tornado）。
* Python coroutine 只会使用一个线程，所以只能利用单核。Goroutine 可以被多个线程调度，可以利用多核。

## References:
* [1][Concurrency is not parallelism](https://talks.golang.org/2012/waza.slide)
* [2][并发编程框架背后的实现，goroutine 背后的系统知识](http://www.infoq.com/cn/articles/knowledge-behind-goroutine)
* [3][CSP wikipedia](https://en.wikipedia.org/wiki/Communicating_sequential_processes)
* [4][Hoare 的 CSP 论文](http://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf)
* [5][Go 对 Hoare CSP 论文中实例的实现](https://github.com/thomas11/csp/blob/master/csp.go)
* [6][Why build concurrency on the ideas of CSP?](https://golang.org/doc/faq#csp)
* [7][The Go scheduler](http://morsmachine.dk/go-scheduler)
* [8][How goroutines work](http://blog.nindalf.com/how-goroutines-work/)
* [9][Goroutines vs OS threads](https://groups.google.com/forum/#!topic/Golang-nuts/j51G7ieoKh4)
* [10][Why goroutines instead of threads?](https://golang.org/doc/faq#goroutines)
* [11][Analysis of the Go runtime scheduler](http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf)
* [12][golang 的 goroutine 是如何实现的？ - 知乎](https://www.zhihu.com/question/20862617/answer/18582460)
