---
title: "Visualizing Go Scheduler Events at Runtime"
date: 2025-12-01T09:10:16+08:00
draft: false
tags: ['Go', 'eBPF', 'not-so-n00b']
---

Being a Go dev for some time, my curiosity in Go's runtime has been gradually built up to the point where I long for more mental grasp than roughly knowing a concept or two. With this in mind, I spent the past few months building [go-slowmo](https://kailunli.me/go-slowmo), some simple visualization around Go's GMP scheduling model, which hopefully provides myself and others another way to look into that part of the Go runtime.

## The idea

There are a handful of utilities out there already, including Go's native tooling - Go's runtime/trace package, combined with the trace file analysis and visualization tool (["go tool trace"](https://pkg.go.dev/cmd/trace)), can give lots of insights on the activity of Go scheduler. However, the existing toolings are (reasonabily) putting more focus on profiling-related perspectives, and maybe for slow starters (like myself) who prefer to first develop their own understanding on the concepts before using tools that are built around the concepts, there could be something that helps bridge the little gap.

There are already learning materials out there for this "bridging cognitive gap" purpose as well - blog posts, YouTube videos, etc. But when I came across [loupe](https://github.com/latentflip/loupe) when trying to pick up JS event loop a while back, I felt like building something similar (i.e. visualization at runtime with the slowed-down code execution displayed side by side) can probably help with the intuitiveness a bit more, and I therefore went for it.

## The big picture

To instrument programs written in a compiled language like Go, some lower-level power needs to be leveraged, and [eBPF](https://docs.ebpf.io/) was picked up for my case. The overall architecture can be simpified as the following:

![alt text][architecture]

[architecture]: /images/go-slowmo/architecture.png "overall architecture"

There's perhaps only one thing that's not straightforward from the diagram and worth pointing out: loading the eBPF program and maps is a pretty priviledged operation and therefore requires some extra Linux capabilities compared to regular applications. This led to some complications for deployment - I really needed to "own" a host (instead of being a "tenant" on a host) to run the instrumentation, and the instrumented program needs to be further separated from the instrumentor into a sidecar container to avoid being too priviledged. Since I can only afford running the entire thing serverlessly, and most serverless platform only allow running application in strictly sandboxed environment, I had to further delegate the heavy lifting of actually running the instrumentd program to some cloud VM platform (which is not as lightweighted as wished), leaving the serverless platform with only some simple proxy-like work.

## The hacks

Along the way there were some interesting hacks implemented to make things work, and some of the most notable ones are shared below.

### Accessing Go struct field offsets

To instrument Go scheduler, the instrumentor inevitably had to be coupled with some internal structures of Go's runtime package and to be aware of some struct field offsets in memory. More importantly, it aims to work with as many future Go versions as possible without changing its code (something like forward compatibility). Two major approaches are applied to meet this goal:

1. Only a small amount of core internal runtime structures and functions are peeked at by the instrumentor (e.g. runtime.schedule(), runtime.gopark(), p.runq).
2. Instead of being directly hardcoded in the instrumentor code, the struct field offsets are generated "dynamically" from the compiled ELF's DWARF section.

### Attaching "uretprobe" to Go functions

There were times I needed to attach instrumentor probes to the end of some internal runtime functions. Due to the dynamic nature of Go's function stack, instead of directly attaching [uretprobe](https://man7.org/linux/man-pages/man2/uretprobe.2.html) to a target function, PCs of a Go function's RET instructions need to be manually retrieved and attached probes to. This again involves some work on analyzing the compiled ELF. A typical routine that finds a function's return PCs using Go's standard [debug/elf](https://pkg.go.dev/debug/elf) package can be illustrated as below:

![alt text][finding-return-pc]

[finding-return-pc]: /images/go-slowmo/uretprobe.png "finding return PCs of a function"

### Bypassing preemption

To look at all the moving parts of Go scheduler, the execution of the program apparently has to be slowed down. But when a goroutine is slowed down (even with some kernel space mechanism), it looks like this goroutine is taking up a long consecutive period of CPU time from the scheduler's perspective, leading to the sysmon to preempt the goroutine to avoid starving other goroutines. Simply put, the sysmon records the tick when a goroutine was scheduled last time and compare it with the current tick to decide how long the goroutine has been taking up the CPU. So to hack with this preemption behaviour, these scheduling related ticks are manipulated by the instrumentor to make a goroutine doesn't look like a greedy CPU time eater.

## Next steps

There are by far two primary limitations for go-slowmo. First, though the whole thing is expected to be running in serverless manner, a "warm pool" of VMs to run instrumented programs is not kept, meaning that the application is not genuinely serverless and the cold start time of VM is visible to users. Second, the vsualized scheduler actions are still not complete - other important scheduler events like network polling and preemption (and therefore the global runq) are missing. The capacity issue is unfortunately unsolvable at this moment, but for the completeness issue, I'll be actively work on it in order to present a more complete and more correct illusion of the scheduler. Last but not least, any feedback is welcome and I sincerely appreciate that in advance.