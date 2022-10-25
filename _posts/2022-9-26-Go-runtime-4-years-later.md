---
layout: post
title: Go runtime：4year later
subtitle: An overview of the evolution of the go language
tags: [test]
comments: true
---

[The Go Blog**](https://go.dev/blog/)

**Go runtime: 4 years later**

*Michael Knyszek*

*26 September 2022*

Since our [last blog post about the Go GC in 2018](https://go.dev/blog/ismmkeynote) the Go GC, and the Go runtime more broadly, has been steadily improving. We’ve tackled some large projects, motivated by real-world Go programs and real challenges facing Go users. Let’s catch you up on the highlights!



**What’s new?**

- sync.Pool, a GC-aware tool for reusing memory, has a [lower latency impact](https://go.dev/cl/166960) and [recycles memory much more effectively](https://go.dev/cl/166961) than before. (Go 1.13)



- The Go runtime returns unneeded memory back to the operating system [much more proactively](https://go.dev/issue/30333), reducing excess memory consumption and the chance of out-of-memory errors. This reduces idle memory consumption by up to 20%. (Go 1.13 and 1.14)

（更主动，减少过多的内存消耗和内存不足带来的错误的概率）

- The Go runtime is able to preempt goroutines more readily in many cases, reducing stop-the-world latencies up to 90%. [Watch the talk from Gophercon 2020 here.](https://www.youtube.com/watch?v=1I1WmeSjRSw) (Go 1.14)

（在许多情况下更稳定地抢占goroutine）

- The Go runtime [manages timers more efficiently than before](https://go.dev/cl/171883), especially on machines with many CPU cores. (Go 1.14)

Go 运行时[比以前更有效地管理计时器](https://go.dev/cl/171883)，尤其是在具有许多 CPU 内核的机器上。



- Function calls that have been deferred with the defer statement now cost as little as a regular function call in most cases. [Watch the talk from Gophercon 2020 here.](https://www.youtube.com/watch?v=DHVeUsrKcbM) (Go 1.14)

在大多数情况下，使用语句延迟的函数调用defer现在与常规函数调用一样少

- The memory allocator’s slow path [scales](https://go.dev/issue/35112) [better](https://go.dev/issue/37487) with CPU cores, increasing throughput up to 10% and decreasing tail latencies up to 30%, especially in highly-parallel programs. (Go 1.14 and 1.15)

内存分配器的慢速路径可以更好地与 CPU 内核一起扩展 ，将吞吐量提高多达 10%，并将尾部延迟降低多达 30%，尤其是在高度并行的程序中

- Go memory statistics are now accessible in a more granular, flexible, and efficient API, the [runtime/metrics](https://pkg.go.dev/runtime/metrics) package. This reduces latency of obtaining runtime statistics by two orders of magnitude (milliseconds to microseconds). (Go 1.16)

Go 内存统计信息现在可以在更精细、灵活和高效的 API 中访问，即runtime/metrics 包。这将获取运行时统计信息的延迟降低了两个数量级（毫秒到微秒）

- The Go scheduler spends up to [30% less CPU time spinning to find new work](https://go.dev/issue/43997). (Go 1.17)

Go 调度程序花费的 CPU 时间减少多达30%，以寻找新的工作。（去 1.17）

- Go code now follows a [register-based calling convention](https://go.dev/issues/40724) on amd64, arm64, and ppc64, improving CPU efficiency by up to 15%. (Go 1.17 and Go 1.18)

- Go 代码现在在 amd64、arm64 和 ppc64 上遵循基于[寄存器的调用约定](https://go.dev/issues/40724)，将 CPU 效率提高多达 15%。（去 1.17 和去 1.18）



- The Go GC’s internal accounting and scheduling has been [redesigned](https://go.dev/issue/44167), resolving a variety of long-standing issues related to efficiency and robustness. This results in a significant decrease in application tail latency (up to 66%) for applications where goroutines stacks are a substantial portion of memory use. (Go 1.18)

- Go GC 的内部会计和调度已经过 [重新设计](https://go.dev/issue/44167)，解决了与效率和健壮性相关的各种长期存在的问题。对于 goroutine 堆栈占内存使用量很大一部分的应用程序，这会显着降低应用程序尾部延迟（高达 66%）。（去 1.18）



- The Go GC now limits [its own CPU use when the application is idle](https://go.dev/issue/44163). This results in 75% lower CPU utilization during a GC cycle in very idle applications, reducing CPU spikes that can confuse job shapers. (Go 1.19)

These changes have been mostly invisible to users: the Go code they’ve come to know and love runs better, just by upgrading Go

Go GC 现在[会在应用程序空闲时限制自己的 CPU 使用](https://go.dev/issue/44163)。这导致在非常空闲的应用程序的 GC 周期期间 CPU 利用率降低了 75%，从而减少了可能使作业成形者感到困惑的 CPU 峰值。

这些变化对用户来说几乎是不可见的：他们已经了解和喜爱的 Go 代码运行得更好，只需升级 Go。

**A new knob**



With Go 1.19 comes an long-requested feature that requires a little extra work to use, but carries a lot of potential: [the Go runtime’s soft memory limit](https://pkg.go.dev/runtime/debug#SetMemoryLimit).

Go 1.19 带来了一个需求已久的功能，需要一些额外的工作才能使用，但具有很大的潜力：[Go 运行时的软内存限制](https://pkg.go.dev/runtime/debug#SetMemoryLimit)。



For years, the Go GC has had only one tuning parameter: GOGC. GOGC lets the user adjust [the trade-off between CPU overhead and memory overhead made by the Go GC](https://pkg.go.dev/runtime/debug#SetGCPercent). For years, this “knob” has served the Go community well, capturing a wide variety of use-cases.

多年来，Go GC 只有一个调优参数：GOGC. GOGC允许用户调整[Go GC 造成的 CPU 开销和内存开销之间的权衡](https://pkg.go.dev/runtime/debug#SetGCPercent)。多年来，这个“旋钮”一直很好地服务于 Go 社区，捕获了各种各样的用例。



The Go runtime team has been reluctant to add new knobs to the Go runtime, with good reason: every new knob represents a new *dimension* in the space of configurations that we need to test and maintain, potentially forever. The proliferation of knobs also places a burden on Go developers to understand and use them effectively, which becomes more difficult with more knobs. Hence, the Go runtime has always leaned into behaving reasonably with minimal configuration.

Go 运行时团队一直不愿意向 Go 运行时添加新的旋钮，这是有充分理由的：每个新旋钮都代表了我们需要测试和维护的配置空间中的一个新*维度，可能是永远的。*旋钮的激增也给 Go 开发人员带来了有效理解和使用它们的负担，随着旋钮的增多，这变得更加困难。因此，Go 运行时总是倾向于以最少的配置合理地运行。

So why add a memory limit knob?



Memory is not as fungible as CPU time. With CPU time, there’s always more of it in the future, if you just wait a bit. But with memory, there’s a limit to what you have.



The memory limit solves two problems.

那么为什么要添加内存限制旋钮呢？



内存不如 CPU 时间可替代。有了 CPU 时间，如果您稍等片刻，将来总会有更多的时间。但是有了记忆，你所拥有的东西是有限的。

内存限制解决了两个问题。



The first is that when the peak memory use of an application is unpredictable, GOGC alone offers virtually no protection from running out of memory. With just GOGC, the Go runtime is simply unaware of how much memory it has available to it. Setting a memory limit enables the runtime to be robust against transient, recoverable load spikes by making it aware of when it needs to work harder to reduce memory overhead.

第一个是当应用程序的内存使用峰值不可预测时， GOGC仅靠它几乎无法防止内存耗尽。使用 just GOGC，Go 运行时根本不知道它有多少可用内存。设置内存限制使运行时能够知道何时需要更努力地工作以减少内存开销，从而对瞬态、可恢复的负载峰值保持稳健。

The second is that to avoid out-of-memory errors without using the memory limit, GOGC must be tuned according to peak memory, resulting in higher GC CPU overheads to maintain low memory overheads, even when the application is not at peak memory use and there is plenty of memory available. This is especially relevant in our containerized world, where programs are placed in boxes with specific and isolated memory reservations; we might as well make use of them! By offering protection from load spikes, setting a memory limit allows for GOGC to be tuned much more aggressively with respect to CPU overheads.



二是在不使用内存限制的情况下避免out-of-memory错误， GOGC必须根据内存峰值进行调优，导致更高的GC CPU开销来保持较低的内存开销，即使应用程序不是在内存使用的峰值和有有足够的可用内存。这在我们的容器化世界中尤其重要，程序被放置在具有特定和隔离内存预留的盒子中；我们不妨利用它们！通过提供对负载峰值的保护，设置内存限制可以 GOGC更积极地调整 CPU 开销。



The memory limit is designed to be easy to adopt and robust. For example, it’s a limit on the whole memory footprint of the Go parts of an application, not just the Go heap, so users don’t have to worry about accounting for Go runtime overheads. The runtime also adjusts its memory scavenging policy in response to the memory limit so it returns memory to the OS more proactively in response to memory pressure.

内存限制旨在易于采用且稳健。例如，它限制了应用程序的 Go 部分的整个内存占用，而不仅仅是 Go 堆，因此用户不必担心考虑 Go 运行时开销。运行时还会根据内存限制调整其内存清理策略，以便更**主动**地将内存返回给操作系统以响应内存压力。

But while the memory limit is a powerful tool, it must still be used with some care. One big caveat is that it opens up your program to GC thrashing: a state in which a program spends too much time running the GC, resulting in not enough time spent making meaningful progress.

For example, a Go program might thrash if the memory limit is set too low for how much memory the program actually needs. GC thrashing is something that was unlikely previously, unless GOGC was explicitly tuned heavily in favor of memory use. We chose to favor running out of memory over thrashing, so as a mitigation, the runtime will limit the GC to 50% of total CPU time, even if this means exceeding the memory limit.

但是，虽然内存限制是一个强大的工具，但仍必须谨慎使用。一个重要的警告是它会使您的程序面临 GC 抖动：程序花费太多时间运行 GC 的状态，导致没有足够的时间花在有意义的进展上。例如，如果将内存限制设置得太低而无法满足程序实际需要的内存量，Go 程序可能会崩溃。GC thrashing 是以前不太可能发生的事情，除非GOGC显式调整以支持内存使用。我们选择内存不足而不是抖动，因此作为一种缓解措施，运行时会将 GC 限制为总 CPU 时间的 50%，即使这意味着超出内存限制。



All of this is a lot to consider, so as a part of this work, we released [a shiny new GC guide](https://go.dev/doc/gc-guide), complete with interactive visualizations to help you understand GC costs and how to manipulate them.

所有这些都需要考虑很多，因此作为这项工作的一部分，我们发布[了一个闪亮的新 GC 指南](https://go.dev/doc/gc-guide)，其中包含交互式可视化，以帮助您了解 GC 成本以及如何操作它们。

**结论**



**Conclusion**

Try out the memory limit! Use it in production! Read the [GC guide](https://go.dev/doc/gc-guide)!

We’re always looking for feedback on how to improve Go, but it also helps to hear about when it just works for you. [Send us feedback](https://groups.google.com/g/golang-dev)!

**Previous article: **[Go Developer Survey 2022 Q2 Results](https://go.dev/blog/survey2022-q2-results)

[**Blog Index**](https://go.dev/blog/all)



来试试内存限制！在生产中使用它！阅读[GC 指南](https://go.dev/doc/gc-guide)！

我们一直在寻找有关如何改进 Go 的反馈，但它也有助于了解它何时适合您。 [给我们反馈](https://groups.google.com/g/golang-dev)！

**上一篇文章：**[Go 开发者调查 2022 Q2 结果](https://go.dev/blog/survey2022-q2-results)



来自 <[https://go.dev/blog/go119runtime](https://go.dev/blog/go119runtime)\>







