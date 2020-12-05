Go给我们提供了一个工具`Trace`，可以在运行时启用`Trace`, 并获得程序执行情况的详细视图。

应该怎么使用`Trace`呢？一般有下面三种使用方式

- 运行`go test`的时候，带上`-trace`参数标记, `go test -trace=trace.out`
- 从`pprof`中获取实时`trace, `import _ "net/http/pprof"`
- 在代码中编码嵌入`trace`,

代码中使用, 我写了一个简单的例子：

```go
package main

import (
	"io/ioutil"
	"log"
	"os"
	"runtime/trace"
	"sync"
)

// 一个简单的例子去演示如何在代码中使用trace包
func main() {
	// 1. 创建trace持久化的文件句柄
	f, err := os.Create("trace.out")
	if err != nil {
		log.Fatalf("failed to create trace output file: %v", err)
	}
	defer func() {
		if err := f.Close(); err != nil {
			log.Fatalf("failed to close trace file: %v", err)
		}
	}()

	// 2. trace绑定文件句柄
	if err := trace.Start(f); err != nil {
		log.Fatalf("failed to start trace: %v", err)
	}
	defer trace.Stop()

	// 下面就是你的监控的程序
	// 我简单写了一个文件读写
	var wg sync.WaitGroup
	wg.Add(2)

	// 一个协程用来读文件
	go func() {
		defer wg.Done()
		ioutil.ReadFile(`mxc.txt`)
	}()

	// 写文件协程
	go func() {
		defer wg.Done()
		ioutil.WriteFile(`mxc_code.txt`, []byte(`码小菜`), 0644)
	}()

	wg.Wait()
}

```

运行命令

![593045dc21dbb45adbba060a8183edf1](/Users/didi/Documents/blog/Go中Trace包的使用.assets/593045dc21dbb45adbba060a8183edf1.png)

会看到生成一个`trace.out`的文件，打开命令

![9887ec50e39400d49bc14f7b6fdfedb7](/Users/didi/Documents/blog/Go中Trace包的使用.assets/9887ec50e39400d49bc14f7b6fdfedb7.png)

这时候会打开一个浏览器窗口，可以看到

![261ab8fd5279e648bdf9d35f776724ac](/Users/didi/Documents/blog/Go中Trace包的使用.assets/261ab8fd5279e648bdf9d35f776724ac.png)

## 工作原理

`Trace`的工作流程非常简单, 标准库会记录Go运行期间的每个事件，例如内存分配, 垃圾收集器(gc)的所有阶段, goroutine何时运行，暂停等等，并将这些信息格式化以供以后显示使用。

需要注意的是, 在开始记录之前，`runtime`会`STW`，并对当前`goroutines`的状态进行快照。

![Image for post](/Users/didi/Documents/blog/Go中Trace包的使用.assets/1*D7gZ3oeiCnbF5DR3J6qhVw.png)

`stw`后，收集的事件会放入`trace`的一个缓冲区，在达到缓冲区最大容量时刷新。下面是整个流程图

![Image for post](/Users/didi/Documents/blog/Go中Trace包的使用.assets/1*Mg1yenwpqb0__HBEu5dWnw.png)

`tracer`现在就可以将上面收集的信息`dump`到输出。所以，当`trace`开始时，Go会专门为`trace`启动一个`goroutine`。 这个`goroutine`会在数据可用时`dump`记录的数据到输出，在没有数据时park住。 如下图：
							![Image for post](/Users/didi/Documents/blog/Go中Trace包的使用.assets/1*Ds-M17xBw8uTAtp8zcjXlA.png)

现在的流程就非常清晰了

一旦生成了跟踪信息，就可以用命令`go tool trace output.out`来完成可视化, 下图是我刚开始那个简单程序的一个trace图

![image-20201205212953602](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205212953602.png)

上面的代码相对简单，看不到更多的关于gc的信息，下面我写了一个触发gc的程序

```go
package main

import (
	"log"
	"os"
	"runtime/trace"
	"sync"
)

func main() {
	// 1. 创建trace持久化的文件句柄
	f, err := os.Create("trace.out")
	if err != nil {
		log.Fatalf("failed to create trace output file: %v", err)
	}
	defer func() {
		if err := f.Close(); err != nil {
			log.Fatalf("failed to close trace file: %v", err)
		}
	}()

	// 2. trace绑定文件句柄
	if err := trace.Start(f); err != nil {
		log.Fatalf("failed to start trace: %v", err)
	}
	defer trace.Stop()

	// 下面就是你的监控的程序
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		for i := 0; i < 100; i++ {
			_ = make([]int, 0, 20000)
		}
	}()

	go func() {
		defer wg.Done()
		for i := 0; i < 100; i++ {
			_ = make([]int, 0, 10000)
		}
	}()

	wg.Wait()
}

```

图示很直接。与`gc`相关的信息，如下图
![image-20201205224751025](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205224751025.png)

下面是一个大致的描述

- **STW** 是gc中的两个"停止世界"的阶段。 在这两个阶段中，`goroutine`会停止运行。

![image-20201205230137415](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205230137415.png)

- **GC（idle）** 指没有标记内存时的goroutine。

![image-20201205230214993](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205230214993.png)

- **MARK ASSIST** 在分配内存过程中重新标记内存(mark the memory)的goroutine。

![image-20201205230326397](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205230326397.png)

- **SWEEP** 垃圾回收

![image-20201205230621032](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205230621032.png)

- **GXX runtime.gcBgMarkWorker** 是帮助标记内存的专用后台goroutine。

![image-20201205230942287](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205230942287.png)

然而，有些Trace并不容易理解:

- `proc start`当一个逻辑处理器(P)与一个线程绑定时被调用。 也就是启动新线程或从系统调用恢复

![image-20201205231716219](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205231716219.png)

- `proc stop`当线程与当前处理器(P)分离时，将调用`proc stop`。当线程发生系统调用被阻塞或线程退出时，就会发生这种情况。

![image-20201205231753587](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205231753587.png)

- `goroutine`进行系统调用时，将调用`syscall`

![image-20201205232054553](/Users/didi/Documents/blog/Go中Trace包的使用.assets/image-20201205232054553.png)



注意：Go既可以定义和可视化自己的Trace以及也可以追踪标准库中的Trace

## 用户Traces

用户级别可以定义的Trace具有两个层次级别：

- 在任务的最高层，有开始和结束
- 在该区域的子级别上

这里有一个简单的例子:

```go
package main

import (
	"context"
	"io/ioutil"
	"log"
	"os"
	"runtime/trace"
	"sync"
)

// 一个简单的例子去演示如何在代码中使用trace包
func main() {
	// 1. 创建trace持久化的文件句柄
	f, err := os.Create("trace.out")
	if err != nil {
		log.Fatalf("failed to create trace output file: %v", err)
	}
	defer func() {
		if err := f.Close(); err != nil {
			log.Fatalf("failed to close trace file: %v", err)
		}
	}()

	// 2. trace绑定文件句柄
	if err := trace.Start(f); err != nil {
		log.Fatalf("failed to start trace: %v", err)
	}
	defer trace.Stop()

	ctx, task := trace.NewTask(context.Background(), "main start")
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		r := trace.StartRegion(ctx, "reading file")
		defer r.End()

		ioutil.ReadFile(`n1.txt`)
	}()

	go func() {
		defer wg.Done()
		r := trace.StartRegion(ctx, "writing file")
		defer r.End()

		ioutil.WriteFile(`n2.txt`, []byte(`42`), 0644)
	}()

	wg.Wait()
	defer task.End()
}
```

可以通过菜单中的**用户自定义任务**直接从工具中可视化这些Trace：

![Image for post](/Users/didi/Documents/blog/Go中Trace包的使用.assets/1*R-jVhTlOrFdAX_bwdH9Y-Q.png)

也可以记录一些其他的信息到任务：

```go
ctx, task := trace.NewTask(context.Background(), "main start")
trace.Log(ctx, "category", "I/O file")
trace.Log(ctx, "goroutine", "2")
```

这些日志将在设置任务的`goroutine`下找到：

![Image for post](/Users/didi/Documents/blog/Go中Trace包的使用.assets/1*hJf5KEs2Ny5F1CLaeg-y5w.png)

一个简单的性能测试，可以帮助理解`Trace`对应用程序的影响。 一个将带有`-trace`标志，而另一个则不带`-trace`。 以下是带有`ioutil.ReadFile（）`函数的性能测试的结果，该函数会生成大量事件：

```go
name         time/op
ReadFiles-8  48.1µs ± 0%
name         time/op
ReadFiles-8  63.5µs ± 0%  // with tracing
```

在这种情况下，性能影响约为35％，并且可能和应用有关系, 你可以根据你的应用性能来决定要用不用

