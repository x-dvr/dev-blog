---
author: ["Denis Rodin"]
title: "Fan-In: Can we Go faster?"
date: "2025-05-20"
description: "Performance comparison of different fan-in implementations in Golang"
# summary: ""
tags: ["Go", "Performance"]
categories: ["Golang"]
ShowToc: true
TocOpen: false
draft: false
---

## What is it all about?

Fan-in is a pretty common concurrency pattern in Go, often used to aggregate results from multiple goroutines. For example, when implementing a worker pool to process jobs concurrently, fan-in can used to collect results from all workers into a single channel. Or imagine you are writing a system to aggregate logs or metrics from different sources. Out of the box Go provides all required instruments to easily build fan-in implementation. What we effectively need to do is to write a function to merge several channels into one.

The most standard implementation can look like (you can find it in many articles/guides online):
```go {linenos=true}
func MergeCanonical[T any](inputs ...<-chan T) <-chan T {
	outCh := make(chan T)
	wg := &sync.WaitGroup{}
	wg.Add(len(inputs))

	// create data transfer goroutine for each input channel
	for _, ch := range inputs {
		go func() {
			defer wg.Done()
			for val := range ch {
				outCh <- val
			}
		}()
	}

	// wait for all goroutines to finish, then close the channel
	go func() {
		wg.Wait()
		close(outCh)
	}()

	return outCh
}
```

Here we create one goroutine per channel to read values from it until it is closed. And we use `sync.WaitGroup` to close the output channel safely after all goroutines are finished. This code looks clean, idiomatic, and works quite efficiently. But person with background in more low-level languages like `c`, `rust` or `zig` can perceive this implementation as a bit wasteful. Spawning green thread for each source? That means allocating memory to at least hold stack of all of them. So let's not resist the urge to explore whether there's a faster or more efficient implementation 🙂.

## In the search of most optimal implementation  

Let's start by drafting a simple case with three input channels. In this case we only need one `select` statement:

```go {linenos=true}
func MergeThree[T any](ch1 <-chan T, ch2 <-chan T, ch3 <-chan T) <-chan T {
	outCh := make(chan T)

	go func() {
		// run loop until all input channels are closed
		for ch1 != nil || ch2 != nil || ch3 != nil {
			select {
			case val, ok := <-ch1:
				if !ok {
					// set the channel to nil to disable this select case
					ch1 = nil
					continue
				}
				outCh <- val
			case val, ok := <-ch2:
				if !ok {
					ch2 = nil
					continue
				}
				outCh <- val
			case val, ok := <-ch3:
				if !ok {
					ch3 = nil
					continue
				}
				outCh <- val
			}
		}
		close(outCh)
	}()

	return outCh
}
```
From the first glance this implementation is at least more efficient in way that it does not spawn separate goroutine to read each input channel. But this way we can only handle fixed amount of input channels.

Can we extrapolate this implementation to dynamic amount of input channels not known at compile time? 🤔 - Yes we can! And for that we will use reflection. Yes, I know, I know, using reflection can really slow down your program, but shouldn't we at least try?

```go {linenos=true}
func MergeReflect[T any](inputs ...<-chan T) <-chan T {
	outCh := make(chan T, len(inputs))
	cases := make([]reflect.SelectCase, len(inputs))

	// create reflect.SelectCase for each input channel
	for i, ch := range inputs {
		cases[i] = reflect.SelectCase{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(ch)}
	}
	go func() {
		defer close(outCh)

		// keep selecting until all channels are closed
		for len(cases) > 0 {
			idx, val, ok := reflect.Select(cases)
			if !ok {
				// channel is closed, remove it's case
				cases = append(cases[:idx], cases[idx+1:]...)
				continue
			}
			v, _ := val.Interface().(T)
			outCh <- v
		}
	}()

	return outCh
}
```

Now that we have two implementations we can run some benchmarks to compare them[^1]. We will do the benchmarks for three use-cases:
1) "worker pool simulation" - small amount of sources (equal to number of CPUs);
1) "metrics collection simulation" - moderate amount of sources;
1) "huge event based system" - large amount of sources.
For simplicity we will use `int` data type for all three use-cases[^2].

Let's start by preparing "testing infrastructure":
```go {linenos=true}
// helper function to create required amount of channels
func setupSources(srcCount int) ([]chan<- int, []<-chan int) {
	writable := make([]chan<- int, srcCount)
	readable := make([]<-chan int, srcCount)
	for idx := range srcCount {
		ch := make(chan int, 5)
		writable[idx] = ch
		readable[idx] = ch
	}

	return writable, readable
}

func closeAll(ch []chan<- int) {
	for _, c := range ch {
		close(c)
	}
}

// send "count" messages, distributing equally across channels
func send(ch []chan<- int, count int) {
	goroutineCount := len(ch)
	// amount of messages sent by each goroutine
	batchSize := count / (goroutineCount - 1)
	lastBatch := count - batchSize*(goroutineCount-1)
	for i := range goroutineCount {
		size := batchSize
		if i == goroutineCount-1 {
			size = lastBatch
		}
		go func() {
			for k := range size {
				ch[i] <- k
			}
		}()
	}
}

// read "count" messages from resulting channel
func read(ch <-chan int, count int) int {
	var sink int
	for i := 0; i < count; i++ {
		sink = <-ch
	}
	return sink
}
```

All our benchmarks will look the same, the only difference is amount of input channels and exact fan-in method we are testing. For worker simulation we will  use amount of input channels equal to `runtime.NumCPU()`, for metrics simulation we will use 100 input channels, and to simulate huge event system we will use 1000 input channels. Amount of messages to be sent is always ten times the amount of input channels, so each channel will send 10 messages.

```go {linenos=true}
func BenchmarkCanonical(b *testing.B) {
	srcCount := runtime.NumCPU()
	msgCount := runtime.NumCPU() * 10
	w, r := setupSources(srcCount)

	out := fanin.MergeCanonical(r...)

	for b.Loop() {
		send(w, msgCount)
		sink = read(out, msgCount)
	}

	closeAll(w)
}

func BenchmarkReflect(b *testing.B) {
	srcCount := runtime.NumCPU()
	msgCount := runtime.NumCPU() * 10
	w, r := setupSources(srcCount)

	out := fanin.MergeReflect(r...)

	for b.Loop() {
		send(w, msgCount)
		sink = read(out, msgCount)
	}

	closeAll(w)
}
```

So let's look at our first results:
```
goos: linux
goarch: amd64
pkg: github.com/x-dvr/go_experiments/fanin
cpu: Intel(R) Core(TM) i7-10870H CPU @ 2.20GHz
BenchmarkWorkerPoolCanonical-16                    34435             34947 ns/op             779 B/op         16 allocs/op
BenchmarkWorkerPoolReflect-16                       4508            241188 ns/op          174899 B/op       3216 allocs/op
BenchmarkMetricsCanonical-16                        4651            261056 ns/op            4885 B/op        100 allocs/op
BenchmarkMetricsReflect-16                           122           9791728 ns/op         7368072 B/op     104134 allocs/op
BenchmarkHugeSourceCountCanonical-16                 309           3888395 ns/op           50734 B/op       1015 allocs/op
BenchmarkHugeSourceCountReflect-16                     1        1221015673 ns/op        694551600 B/op  10042288 allocs/op
```
That is quite a disappointing result for potential fans of reflection. In the worker pool use case, the canonical implementation outperformed the reflection-based one by almost 7 times; in the metrics use case, by 37 times; and in the huge event system — by a whopping 314 times! What's also hard to miss is the number of allocations made by the reflection-based implementation. In canonical implementation we have allocations for each sending goroutine (I couldn't find a cleaner way to measure message passing performance). But in reflection-based version we have amount of allocations proportional to the product of amount input channels and amount of sent messages. If we run the reflection benchmark with the option to collect a memory profile, we’ll see that 99% of the memory is allocated inside the `reflect.Select` function. So on every call it does an allocation for each select case in the list.

In order to reduce amount of goroutines required to process input channels we can try two more approaches:
- Split input channels into groups of 4, spawn one goroutine for each group and use select operator;
- Spawn only one goroutine with a loop over all input channels using select with `default` case to avoid blocking;

Second variant can be much more wasteful, due to the constant non-blocking iteration over all inputs. But let's see the actual benchmarks before doing any conclusions.

Our "split by four" implementation can look like this:
```go {linenos=true}
func MergeBatch[T any](inputs ...<-chan T) <-chan T {
	inputCount := len(inputs)
	// number of goroutines to spawn
	gCount := inputCount / 4
	// number of channels handled by last goroutine
	leftovers := inputCount - (gCount * 4)

	outCh := make(chan T, inputCount)
	wg := &sync.WaitGroup{}
	wg.Add(gCount)

	// create data transfer goroutine for each 4 input channels
	for i := range gCount {
		go select4(wg, outCh,
			inputs[i*4],
			inputs[i*4+1],
			inputs[i*4+2],
			inputs[i*4+3],
		)
	}

	switch leftovers {
	case 3:
		wg.Add(1)
		go select3(wg, outCh,
			inputs[inputCount-3],
			inputs[inputCount-2],
			inputs[inputCount-1],
		)
	case 2:
		wg.Add(1)
		go select2(wg, outCh,
			inputs[inputCount-2],
			inputs[inputCount-1],
		)
	case 1:
		wg.Add(1)
		go read(wg, outCh, inputs[inputCount-1])
	}

	// wait for all goroutines to finish, then close the channel
	go func() {
		wg.Wait()
		close(outCh)
	}()

	return outCh
}

func select4[T any](
	wg *sync.WaitGroup,
	out chan<- T,
	ch1, ch2, ch3, ch4 <-chan T,
) {
	defer wg.Done()
	// loop while we have at least one open channel
	for ch1 != nil || ch2 != nil || ch3 != nil || ch4 != nil {
		select {
		case val, ok := <-ch1:
			if !ok {
				ch1 = nil
				continue
			}
			out <- val
		case val, ok := <-ch2:
			if !ok {
				ch2 = nil
				continue
			}
			out <- val
		case val, ok := <-ch3:
			if !ok {
				ch3 = nil
				continue
			}
			out <- val
		case val, ok := <-ch4:
			if !ok {
				ch4 = nil
				continue
			}
			out <- val
		}
	}
}

func select3[T any](
	wg *sync.WaitGroup,
	out chan<- T,
	ch1, ch2, ch3 <-chan T,
) {
	defer wg.Done()
	// loop while we have at least one open channel
	for ch1 != nil || ch2 != nil || ch3 != nil {
		select {
		case val, ok := <-ch1:
			if !ok {
				ch1 = nil
				continue
			}
			out <- val
		case val, ok := <-ch2:
			if !ok {
				ch2 = nil
				continue
			}
			out <- val
		case val, ok := <-ch3:
			if !ok {
				ch3 = nil
				continue
			}
			out <- val
		}
	}
}

func select2[T any](wg *sync.WaitGroup, out chan<- T, ch1, ch2 <-chan T) {
	defer wg.Done()
	// loop while we have at least one open channel
	for ch1 != nil || ch2 != nil {
		select {
		case val, ok := <-ch1:
			if !ok {
				ch1 = nil
				continue
			}
			out <- val
		case val, ok := <-ch2:
			if !ok {
				ch2 = nil
				continue
			}
			out <- val
		}
	}
}

func read[T any](wg *sync.WaitGroup, out chan<- T, in <-chan T) {
	defer wg.Done()
	for val := range in {
		out <- val
	}
}
```
And final "one big loop" implementation is the pretty straightforward:
```go {linenos=true}
func MergeLoop[T any](inputs ...<-chan T) <-chan T {
	outCh := make(chan T, len(inputs))
	total := len(inputs)
	active := total

	go func() {
		// iterate while we have at least one open channel
		for active > 0 {
			for idx := 0; idx < active; idx++ {
				select {
				case val, ok := <-inputs[idx]:
					if !ok {
						// channel is closed, remove it from inputs and decrease active count
						active--
						inputs = append(inputs[:idx], inputs[idx+1:]...)
						continue
					}
					outCh <- val
				default:
				}
			}
		}
		close(outCh)
	}()

	return outCh
}
```

Time for the second round of benchmarks. I removed reflection-based implementation due to it's extreme inefficiency. Results for all remaining implementations, you can see in a table below:
```
BenchmarkWorkerPoolCanonical-16                    33226             35925 ns/op             780 B/op         16 allocs/op
BenchmarkWorkerPoolBatch-16                        24518             48428 ns/op             769 B/op         16 allocs/op
BenchmarkWorkerPoolLoop-16                         12849             93811 ns/op             770 B/op         16 allocs/op

BenchmarkMetricsCanonical-16                        4347            265471 ns/op            4891 B/op        100 allocs/op
BenchmarkMetricsBatch-16                            3796            307656 ns/op            4828 B/op        100 allocs/op
BenchmarkMetricsLoop-16                             5588            205936 ns/op            4885 B/op        101 allocs/op

BenchmarkHugeSourceCountCanonical-16                 300           3988277 ns/op           50906 B/op       1015 allocs/op
BenchmarkHugeSourceCountBatch-16                     433           2691683 ns/op          130797 B/op       2163 allocs/op
BenchmarkHugeSourceCountLoop-16                      825           1447144 ns/op           48899 B/op       1009 allocs/op
```

## Instead of a conclusion

So, what can we learn from all this? First of all, there's more than one way to implement fan-in in Go — but not all approaches are created equal.

The *canonical implementation*, while simple and idiomatic, performs remarkably well across all scenarios. Despite spawning one goroutine per input, it remains the fastest choice for small to moderately sized workloads. Outperforming batched version by 1.3 times, and loop version by 2.6 times. Even in large setups, it held its own, significantly loosing only to loop-based version, proving that sometimes the obvious solution is the right one.

The *reflection-based* approach, though flexible and elegant at first glance, comes at a significant cost. Its poor performance and massive memory allocation make it unsuitable for any performance-critical application. This experiment reaffirms a common Go wisdom: avoid reflection in the hot path.

Our attempt to reduce goroutines with the *batching strategy* (using select over fixed groups of inputs) yielded mixed results. While it slightly outperformed the canonical version in the large-scale test. It fel of short to canonical implementation for small and medium sized workloads. It's a valid tradeoff in extreme situations where goroutine overhead must be minimized, but not a general-purpose win.

Surprisingly, the *loop-based* approach (using a single goroutine with non-blocking selects) performed extremely well for large and medium input sets. Its simple structure and lower pressure on Go scheduler make it an interesting choice when you should avoid spawning many goroutines. In a heavy load scenario it outperformed canonical implementation by almost 2.8 times. However, its inefficiency in smaller setups and potential CPU waste make it a niche tool.

### TL;DR?

- Canonical: Best all-around. Fast, simple, idiomatic. Use it by default.
- Reflection: Avoid for performance-sensitive code.
- Batch Select: A good option for high-scale setups, if you're OK with some extra code complexity.
- Loop with Default: Works in a pinch for tons of channels, but can be CPU-hungry.

At the end of the day, Go gives us several ways to implement fan-in — and it’s up to us to pick the one that fits our needs best. If you're writing highly concurrent systems in Go, don't make assumptions, it's worth benchmarking your own workload.

[^1]: Full source code including benchmarks can be found in [my repository](https://github.com/x-dvr/go_experiments/tree/master/fanin) inside `fanin` directory.
[^2]: I did run benchmarks with more complex structured data types, but this did not change overall picture.