# 线程

```go
线程允许一个程序同时做很多事情
  每个线程串行执行，就像普通的非线程程序一样
  线程共享内存
  每个线程都包含一些每个线程的状态：
    程序计数器，寄存器，堆栈，它在等待什么
```

# 为什么是线程

```go
1. 输入输出并发
    客户端并行向许多服务器发送请求并等待回复。
    服务器处理多个客户端请求；每个请求都可能阻塞。
    在等待磁盘为客户端 X 读取数据时，
      处理来自客户端 Y 的请求。
2. 多核性能
    在多个内核上并行执行代码。
3. 方便
    在后台，每秒检查一次每个工人是否还活着。
```

# 有没有线程的替代品

```go
 是的：在单个线程中编写显式交错活动的代码。
    通常称为“事件驱动”。
  保留有关每个活动的状态表，例如每个客户端请求。
  一个“事件”循环：
    检查每个活动的新输入（例如来自服务器的回复到达），
    为每个活动执行下一步，
    更新状态。
  事件驱动让你获得 I/O 并发，
    并消除线程成本（可能很大），
    但没有获得多核加速，
    并且编程很痛苦。
```

# 线程面临的挑战

```go
  1. 安全地共享数据
    如果两个线程同时做 n = n + 1 会怎样？
      或者一个线程读取而另一个线程递增？
    这是一场“比赛”——通常是一个错误
    -> 使用锁（Go 的 sync.Mutex）
    -> 或避免共享可变数据
  2. 线程之间的协调
    一个线程正在生产数据，另一个线程正在使用它
      消费者如何等待（并释放 CPU）？
      生产者如何唤醒消费者？
    -> 使用 Go 频道或 sync.Cond 或 sync.WaitGroup
  3. 僵局
    通过锁和/或通信（例如 RPC 或 Go 通道）循环
```

# 示例:网络爬虫

## 什么是网络爬虫

```go
目标：获取所有网页，例如提供给索引器
  你给它一个起始网页
  它递归地遵循所有链接
  但不要多次获取给定页面
    不要陷入循环
```

## 爬虫面临的挑战

```go
  1. 利用 I/O 并发性
    网络延迟比网络容量更受限制
    同时获取多个 URL
      增加每秒获取的 URL
    => 使用线程进行并发
  2. 仅获取每个 URL *一次*
    避免浪费网络带宽
    善待远程服务器
    => 需要记住访问了哪些 URL
  3. 知道什么时候完成
```

## 解决方案一 串行爬虫

```go
通过递归串行调用执行深度优先探索
  “获取”地图避免重复，打破循环
    单个映射，通过引用传递，调用者看到被调用者的更新
  但是：一次只获取一页——慢
    我们可以在 Serial() 调用前面加上一个“go”吗？
    让我们试试看……发生了什么？
```

### 代码

```go
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
```



## 解决方案二 ConcurrentMutex crawler

```go
为每个页面获取创建一个线程
    多并发，获取率更高
  "go func" 创建一个 goroutine 并开始运行
    func... 是一个“匿名函数”
  线程共享“获取的”地图
    所以只有一个线程会获取任何给定的页面
```

### 为什么是互斥锁?

```go
1. 一个理由：
      两个线程使用相同的 URL 同时调用 ConcurrentMutex()
        由于两个不同的页面包含指向相同 URL 的链接
      T1 读取 fetched[url]，T2 读取 fetched[url]
      两者都看到尚未获取 url（已经 == false）
      两者都取，这是错误的
      互斥体导致一个等待，而另一个同时检查和设置
        所以只有一个线程已经看到==false
      我们说“锁保护数据”
        但不是 Go 不会强制锁和数据之间有任何关系！
      锁定/解锁之间的代码通常称为“临界区”
2. 另一个原因：
      在内部，map 是一个复杂的数据结构（树？可扩展哈希？）
      并发更新/更新可能会破坏内部不变量
      并发更新/读取可能会使读取崩溃
```

###  ConcurrentMutex 爬虫如何决定它完成了？

```go
sync.WaitGroup
 Wait() 等待所有 Add() 被 Done() 平衡
      即等待所有子线程完成
    [图表：goroutines 树，覆盖在循环 URL 图上]
    树中的每个节点都有一个 WaitGroup
```

### 代码

```go
//
// Concurrent crawler with shared state and Mutex
//

type fetchState struct {
	mu      sync.Mutex
	fetched map[string]bool
}

func ConcurrentMutex(url string, fetcher Fetcher, f *fetchState) {
	f.mu.Lock()
	already := f.fetched[url]
	f.fetched[url] = true
	f.mu.Unlock()

	if already {
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
			defer done.Done()
			ConcurrentMutex(u, fetcher, f)
		}(u)
	}
	done.Wait()
	return
}

func makeState() *fetchState {
	f := &fetchState{}
	f.fetched = make(map[string]bool)
	return f
}
```





问: 这个爬虫可以创建多少并发线程？



## 解决方案三 ConcurrentChannel 爬虫

### 1.a Go channel:

```go
    a channel is an object
      ch := make(chan int)
	a channel lets one thread send an object to another thread
	ch <- x
	  发送者等到某个 goroutine 收到
	y := <- ch
	for y := range ch
	  接收者等到某个 goroutine 发送
channels both communicate and synchronize(通信又同步)
    several threads can send and receive on a channel
    channels are cheap
    remember: sender blocks until the receiver receives!
      "synchronous"
      watch out for deadlock
```

### 2. ConcurrentChannel coordinator()

```go
coordinator() creates a worker goroutine to fetch each page
    worker() sends slice of page's URLs on a channel
	multiple workers send on the single channel (多个线程使用同一通道)
    coordinator() reads URL slices from the channel
At what line does the coordinator wait?
    Does the coordinator use CPU time while it waits?
At what line does the coordinator wait?
    Does the coordinator use CPU time while it waits?
Note: there is no recursion(递归) here; instead there's a work list.(工作清单)
  Note: no need to lock the fetched map, because it isn't shared!
  How does the coordinator know it is done?
    Keeps count of workers in n.
Each worker sends exactly one item(项目) on channel.
```

为什么多个线程使用同一通道是安全的?

### 3. Worker thread writes url slice, coordinator reads it, is that a race?

```go
 * worker only writes slice *before* sending
  * coordinator only reads slice *after* receiving
  So they can't use the slice at the same time.
```



### 4. When to use sharing and locks, versus(而不是) channels?

```go
  What makes the most sense depends on how the programmer thinks
    state -- sharing and locks
    communication -- channels
  For the 6.824 labs, I recommend sharing+locks for state,
    and sync.Cond or channels or time.Sleep() for waiting/notification.
```





### 代码

```go
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
```



# Remote Procedure Call (RPC)

```
a key piece(关键部分) of distributed system machinery; all the labs use RPC
  goal: easy-to-program client/server communication
  hide details of network protocols
  convert(转换) data (strings, arrays, maps, &c) to "wire format"(有限格式)
  portability / interoperability(可移植性/互操作性)
```

## RPC message diagram:

```go
 Client             Server
    request--->
       <---response
```

## Software structure

```go
client app        handler fns
	stub(存根) fns         dispatcher(调度程序)
    RPC lib           RPC lib
      net  ------------ net
```



# Go example: kv.go on schedule page











```go
//
//exercise-web-crawler.go
//

type Fetcher interface {
	// Fetch returns the body of URL and
	// a slice of URLs found on that page.
	Fetch(url string) (body string, urls []string, err error)
}

// Crawl uses fetcher to recursively crawl
// pages starting with url, to a maximum of depth.
func Crawl(url string, depth int, fetcher Fetcher) {
	// TODO: Fetch URLs in parallel.
	// TODO: Don't fetch the same URL twice.
	// This implementation doesn't do either:
	if depth <= 0 {
		return
	}
	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("found: %s %q\n", url, body)
	for _, u := range urls {
		Crawl(u, depth-1, fetcher)
	}
	return
}

func main() {
	Crawl("https://golang.org/", 4, fetcher)
}

// fakeFetcher is Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

// fetcher is a populated fakeFetcher.
var fetcher = fakeFetcher{
	"https://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"https://golang.org/pkg/",
			"https://golang.org/cmd/",
		},
	},
	"https://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"https://golang.org/",
			"https://golang.org/cmd/",
			"https://golang.org/pkg/fmt/",
			"https://golang.org/pkg/os/",
		},
	},
	"https://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
	"https://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
}
```



