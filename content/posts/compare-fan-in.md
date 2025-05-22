---
author: ["Denis Rodin"]
title: "Fan-In: How fast can we Go?"
date: "2025-05-20"
# description: "Build your own CI tool in Golang"
# summary: "First article in a series of many, which describes my steps in a journey to build my own CI tool from scratch in Golang"
tags: ["Go"]
categories: ["Performance", "Golang"]
# series: ["Build your own CI"]
ShowToc: true
TocOpen: false
draft: true
---

## What it is all about?

Fan-in is a pretty common concurrency pattern in Go	often used to aggregate results from multiple goroutines. For example, when implementing a worker pool to process jobs concurrently, fan-in can used to collect results from all workers into a single channel. Or imagine you are writing a system to aggregate logs or metrics from different sources. Out of the box Go provides all needed instruments to easily build fan-in implementation. What we effectively need to do is to write a function to merge several channels into one.

The most standard implementation can look like:
```go {linenos=true}
func Merge[T any](chans ...<-chan T) <-chan T {
	var wg sync.WaitGroup
	wg.Add(len(chans))

	outCh := make(chan T)
	for _, ch := range chans {
		go func() {
			defer wg.Done()
			for val := range ch {
				outCh <- val
			}
		}()
	}

	go func() {
		wg.Wait()
		close(outCh)
	}()

	return outCh
}

```

Here we create one goroutine per channel to read values from it until it is closed. And we use `sync.WaitGroup` to close the output channel safely after all goroutines are finished. This code looks clean, idiomatic, and works quite efficiently — but I just can't resist the urge to explore whether there's faster or more efficient implementation 🙂.

## In the search of most optimal implementation  

Let's start by drafting a simple case with three input channels. In this case we only need one `select` statement:

```go {linenos=true}
func MergeThree[T any](ch1 <-chan T, ch2 <-chan T, ch3 <-chan T) <-chan T {
	outCh := make(chan T)
	open1, open2, open3 := true, true, true

	go func() {
		for open1 || open2 || open3 {
			select {
			case val, ok := <-ch1:
				if ok {
					outCh <- val
				} else {
					open1 = false
				}
			case val, ok := <-ch2:
				if ok {
					outCh <- val
				} else {
					open2 = false
				}
			case val, ok := <-ch3:
				if ok {
					outCh <- val
				} else {
					open3 = false
				}
			}
		}
		close(outCh)
	}()

	return outCh
}
```



## Outro

This article is already quite big, so I stop here for now. In the next article we will start working on containerization and scalability.

If you encountered any problems please check, that your code is the same as in my repository[^1].

PS: I would really appreciate any constructive feedback. Please feel free to drop a comment [here](https://www.reddit.com/r/golang/comments/1cq9cnt/build_your_own_ci_system_in_go_part_1/).

[^1]: Complete source code for this article can be found in [my repository](https://github.com/flow-ci/flow-ci/tree/6b8caddc2babd3ecf19e16333f749edc0fae2317) at commit `6b8caddc2babd3ecf19e16333f749edc0fae2317`.