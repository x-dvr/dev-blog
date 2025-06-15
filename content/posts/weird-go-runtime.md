---
author: ["Denis Rodin"]
title: "Weird behavior of Go compiler/runtime"
date: "2025-05-20"
description: "Importing runtime package may heavily affects benchmark results in some cases"
# summary: ""
tags: ["Go", "Performance"]
categories: ["Golang"]
ShowToc: true
TocOpen: false
draft: false
---

## Strange optimization

While I was trying to measure performance impact of spawning lots of goroutines doing CPU-bound workload I encountered interesting side effects in my benchmarks.

Let's pretend we have the following benchmark: we spawn many goroutines, each of which is doing calculation of Fibonacci sequence (example of a CPU-bound task). Each goroutine calculates Fibonacci sequence till 10000s element. We spawn 100000 goroutines in parallel and wait until all of them are finished with the work.
```go {linenos=true}
package main_test

import (
	"sync"
	"testing"
)

var (
	CalcTo   int = 1e4
	RunTimes int = 1e5
)

// artificial sink to avoid unwanted compiler optimizations
var sink int = 0

func workHard(calcTo int) {
	var n2, n1 = 0, 1
	for i := 2; i <= calcTo; i++ {
		n2, n1 = n1, n1+n2
	}
	sink = n1
}

type worker struct {
	wg *sync.WaitGroup
}

func (w worker) Work() {
	workHard(CalcTo)
	w.wg.Done()
}

func Benchmark(b *testing.B) {
	var wg sync.WaitGroup
	w := worker{wg: &wg}

	for b.Loop() {
		wg.Add(RunTimes)
		for j := 0; j < RunTimes; j++ {
			go w.Work()
		}
		wg.Wait()
	}
}
```

Running this benchmark gives us a baseline for performance expectations.
```
goos: linux
goarch: amd64
pkg: github.com/x-dvr/go_experiments
cpu: Intel(R) Core(TM) i7-10870H CPU @ 2.20GHz
Benchmark-16                  52          42988069 ns/op         1816684 B/op     100450 allocs/op
PASS
ok      github.com/x-dvr/go_experiments 2.238s
```

So we have here a bit less than 43ms. Next I decided to remove sink, to check how much faster it will run if compiler optimizes away useless computations.

New code:
```go {linenos=true}
package main_test

import (
	"sync"
	"testing"
)

var (
	CalcTo   int = 1e4
	RunTimes int = 1e5
)

func workHard(calcTo int) {
	var n2, n1 = 0, 1
	for i := 2; i <= calcTo; i++ {
		n2, n1 = n1, n1+n2
	}
}

type worker struct {
	wg *sync.WaitGroup
}

func (w worker) Work() {
	workHard(CalcTo)
	w.wg.Done()
}

func Benchmark(b *testing.B) {
	var wg sync.WaitGroup
	w := worker{wg: &wg}

	for b.Loop() {
		wg.Add(RunTimes)
		for j := 0; j < RunTimes; j++ {
			go w.Work()
		}
		wg.Wait()
	}
}
```

Running benchmark for our "optimized" variant, and we get:
```
goos: linux
goarch: amd64
pkg: github.com/x-dvr/go_experiments
cpu: Intel(R) Core(TM) i7-10870H CPU @ 2.20GHz
Benchmark-16                  16          66662840 ns/op
PASS
ok      github.com/x-dvr/go_experiments 1.069s
```

How can that be, optimized code runs 1.5x slower 😱

## Interesting side effect of importing `runtime` package

If you think this is strange, then what happens next is beyond explanation. Without changing anything, let's just add one definition of exported variable to introduce import of `runtime` package:
```go {linenos=inline,hl_lines=[4,12]}
package main_test

import (
	"runtime"
	"sync"
	"testing"
)

var (
	CalcTo   int = 1e4
	RunTimes int = 1e5
	Why      int = runtime.NumCPU()
)

func workHard(calcTo int) {
	var n2, n1 = 0, 1
	for i := 2; i <= calcTo; i++ {
		n2, n1 = n1, n1+n2
	}
}

type worker struct {
	wg *sync.WaitGroup
}

func (w worker) Work() {
	workHard(CalcTo)
	w.wg.Done()
}

func Benchmark(b *testing.B) {
	var wg sync.WaitGroup
	w := worker{wg: &wg}

	for b.Loop() {
		wg.Add(RunTimes)
		for j := 0; j < RunTimes; j++ {
			go w.Work()
		}
		wg.Wait()
	}
}
```

Running this version of benchmark gives "unexpected" expected results:
```
goos: linux
goarch: amd64
pkg: github.com/x-dvr/go_experiments
cpu: Intel(R) Core(TM) i7-10870H CPU @ 2.20GHz
Benchmark-16                  32          36412238 ns/op
PASS
ok      github.com/x-dvr/go_experiments 1.167s
```

Just adding import of `runtime` enables expected compiler optimizations! 

But why?..