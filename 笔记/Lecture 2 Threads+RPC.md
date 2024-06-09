---
dg-publish: true
---
# 1 Why go
- 对线程和RPC支持度高
- 自带garbage collector，不用考虑垃圾回收
- 运行时开销低

# 2 线程
goroutine介绍：[go进阶(1) -深入理解goroutine并发运行机制-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2227925)
goroutine是go语言的协程，可看作一种轻量级的线程，创建和切换开销很小
goroutine实现并发，但在多核系统上也可以实现并发，Go语言的运行时调度器会自动管理Goroutines，并在多个操作系统线程上调度它们，从而利用多核处理器的优势实现并行执行。

## 2.1 使用多线程的原因
- I/O并发( I/O concurrency)
- 多核并行(multi-core parallelism)，增加吞吐量
- 使用方便

## 2.2 多线程挑战
1. 竞态(race conditions)：多个线程同时修改共享资源时，由于操作的顺序不确定，程序的行为和结果将不可预测。如2个线程同时执行`n=n+1`。
	解决方法：
	1. 不共享内存。在go语言中使用channel
	golang有竞态检测器，在运行程序时添加`-race`标志即可启用
	2. 使用锁
2. 协调(coordination)：线程之间的同步协调问题
3. 死锁


go语言在多线程中的使用：
- [[goroutine同步方法]]：channels / locks + condition variables
- golang逃逸分析：[先聊聊Go的「堆栈」，再聊聊Go的「逃逸分析」。 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/586249256)

# 3 爬虫(Crawler)
 - I/O 并发：爬虫的主要操作是网络I/O请求，而不是CPU计算。网络I/O操作通常需要等待服务器响应，这段等待时间对于CPU来说是空闲时间。使用并发爬虫，当一个goroutine等待网络响应时，其他goroutine可以利用这段时间进行其他网络请求，从而更好地利用CPU资源。
 - Fetch url once：每个url只爬一次
 - exploit parallelism：有多个核心时利用并行
[Go 语言之旅 (go-zh.org)](https://tour.go-zh.org/concurrency/10)

课程上的代码，提供了串行爬虫，使用互斥锁的并发爬虫以及使用通道的并发爬虫3种方法
```go fold
package main

import (
	"fmt"
	
	"sync"
)

//
// Several solutions to the crawler exercise from the Go tutorial
// https://tour.golang.org/concurrency/10
//

//
// Serial crawler
//

func Serial(url string, fetcher Fetcher, fetched map[string]bool) {
	if fetched[url] {
		return
	}
	fetched[url] = true
	urls, err := fetcher.Fetch(url)
	if err != nil {
		return
	}
	for _, u := range urls {
		Serial(u, fetcher, fetched)
	}
	return
}

//
// Concurrent crawler with shared state and Mutex
//

type fetchState struct {
	mu      sync.Mutex
	fetched map[string]bool
}

func (fs *fetchState) testAndSet(url string) bool {
	fs.mu.Lock()
	defer fs.mu.Unlock()
	r := fs.fetched[url]
	fs.fetched[url] = true
	return r
}

func ConcurrentMutex(url string, fetcher Fetcher, fs *fetchState) {
	if fs.testAndSet(url) {
		return
	}
	urls, err := fetcher.Fetch(url)
	if err != nil {
		return
	}
	var done sync.WaitGroup
	for _, u := range urls {
		done.Add(1)
		go func(u string) {
			ConcurrentMutex(u, fetcher, fs)
			done.Done()
		}(u)
	}
	done.Wait()
	return
}

func makeState() *fetchState {
	return &fetchState{fetched: make(map[string]bool)}
}

//
// Concurrent crawler with channels
//

func worker(url string, ch chan []string, fetcher Fetcher) {
	urls, err := fetcher.Fetch(url)
	if err != nil {
		ch <- []string{}
	} else {
		ch <- urls
	}
}

func coordinator(ch chan []string, fetcher Fetcher) {
	n := 1
	fetched := make(map[string]bool)
	for urls := range ch {
		for _, u := range urls {
			if fetched[u] == false {
				fetched[u] = true
				n += 1
				go worker(u, ch, fetcher)
			}
		}
		n -= 1
		if n == 0 {
			break
		}
	}
}

func ConcurrentChannel(url string, fetcher Fetcher) {
	ch := make(chan []string)
	go func() {
		ch <- []string{url}
	}()
	coordinator(ch, fetcher)
}

//
// main
//

func main() {
	fmt.Printf("=== Serial===\n")
	Serial("http://golang.org/", fetcher, make(map[string]bool))

	fmt.Printf("=== ConcurrentMutex ===\n")
	ConcurrentMutex("http://golang.org/", fetcher, makeState())

	fmt.Printf("=== ConcurrentChannel ===\n")
	ConcurrentChannel("http://golang.org/", fetcher)
}

//
// Fetcher
//

type Fetcher interface {
	// Fetch returns a slice of URLs found on the page.
	Fetch(url string) (urls []string, err error)
}

// fakeFetcher is Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) ([]string, error) {
	if res, ok := f[url]; ok {
		fmt.Printf("found:   %s\n", url)
		return res.urls, nil
	}
	fmt.Printf("missing: %s\n", url)
	return nil, fmt.Errorf("not found: %s", url)
}

// fetcher is a populated fakeFetcher.
var fetcher = fakeFetcher{
	"http://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"http://golang.org/pkg/",
			"http://golang.org/cmd/",
		},
	},
	"http://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"http://golang.org/",
			"http://golang.org/cmd/",
			"http://golang.org/pkg/fmt/",
			"http://golang.org/pkg/os/",
		},
	},
	"http://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
	"http://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
}
```

# 4 远程过程调用(RPC)
允许程序调用远程服务器上的函数或过程，就像调用本地函数一样。RPC隐藏了网络通信的复杂性，使分布式应用程序开发变得更加简便。
- client：发起调用请求
- server：执行请求的函数并返回结果

工作流程：
- **客户机调用代理（stub）**：客户机调用一个本地代理函数，代理函数负责将调用参数序列化并发送给远程服务器上的代理函数。
- **服务器接收请求**：服务器端的代理函数接收请求并将参数反序列化，然后调用实际的服务函数。
- **服务器执行函数并返回结果**：服务器执行函数，并将结果返回给代理函数。
- **返回结果传输到客户机**：结果通过网络传输回客户机，客户机端的代理函数接收结果反序列号，将结果返回给调用者。

RPC失败时的语义（RPC semantics under failures）：
- at-least-once：至少执行一次，客户端将自动重试并继续
- at-most-once：至多执行一次，即重复请求不再处理
- Exactly-once：正好执行一次，很难做到


框架：
gRPC：[Quick start | Go | gRPC](https://grpc.io/docs/languages/go/quickstart/)



