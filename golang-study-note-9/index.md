# Golang 性能调优


# Go语言基础

## Golang 性能调优

* 在计算机性能调试领域里, `profiling` 是指对应用程序的画像, 画像就是应用程序使用 CPU 和内存的情况。 Go语言是一个对性能特别看重的语言, 因此语言中自带了 `profiling` 的库。 

* Go语言项目中的性能优化主要有以下几个方面:

  1. CPU profile: - 报告程序的 CPU 使用情况, 按照一定频率去采集应用程序在 CPU 和寄存器上面的数据。

  2. Memory Profile (Heap Profile): - 报告程序的内存使用情况。

  3. Block Profiling: - 报告多个 `goroutine` 不在运行状态的情况, 可以用来分析和查找死锁等性能瓶颈。 

  4. Goroutine Profiling：- 报告多个 `goroutine` 的使用情况, 有哪些 `goroutine`, 它们的调用关系是怎样的。



### 性能调优 - 数据采集

* Go语言内置了获取程序的运行数据的工具, 包括以下两个标准库

  * `runtime/pprof`: - 采集工具型应用运行数据进行分析。

  * `net/http/pprof`: - 采集服务型应用运行时数据进行分析。


* `pprof` - 开启后每隔一段时间（10ms）就会收集下当前的堆栈信息, 获取每个函数占用的CPU以及内存资源, 最后通过对这些采样数据进行分析，形成一个性能分析报告。

* 注意: -  我们只应该在性能测试的时候才在代码中引入pprof。



### runtime/pprof 模块


* CPU性能分析

  * 开启CPU性能分析 - `pprof.StartCPUProfile(w io.Writer)` 

  * 停止CPU性能分析 - `pprof.StopCPUProfile()`

  * 应用执行结束后, 就会生成一个文件, 保存了我们的 `CPU profiling` 数据。得到采样数据之后, 使用`go tool pprof`工具进行CPU性能分析。



* 内存性能分析

  * 开启内存分析 - `pprof.WriteHeapProfile(w io.Writer)` 



### net/http/pprof 模块

* `http.DefaultServeMux` - 默认的 `http.ListenAndServe()` 绑定服务, 只需要在代码中导入 `import _ "net/http/pprof"` 包既可对程序进行分析。

*  使用自定义`Mux`, 需要手动注册 `pprof` 模块的路由规则。

```go
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```


* gin 框架中需要进行分析, 可使用 `github.com/DeanThompson/ginpprof` 。


* 使用 `pprof` 模块 以后自动生成一个 路由组 `/debug/pprof` 包含如下几个路由。

  * `/debug/pprof/profile`: - 访问这个链接会自动进行 `CPU profiling`, 持续 `30s`, 并生成一个文件供下载。

  * `/debug/pprof/heap`: -  `Memory Profiling` 的路径, 访问这个链接会得到一个内存 `Profiling` 结果的文件。

  * `/debug/pprof/block`: - `block Profiling` 的路径。

  * `/debug/pprof/goroutines`: - 运行的 `goroutines` 列表, 以及调用关系。

* 访问地址如: `http://localhost:8080/debug/pprof` 。



### go tool pprof

* 不管是`runtime/pprof` 还是 `net/http/pprof`, 我们使用相应的`pprof`库获取数据之后, 下一步的都要对这些数据进行分析, 我们可以使用`go tool pprof`命令行工具。

* `go tool pprof [binary] [source]`

  * `binary` - 是应用的二进制文件, 用来解析各种符号。

  * `source` - 表示 `profile` 数据的来源, 可以是本地的文件, 也可以是 `http` 地址。


* 注意:  - 获取的 `Profiling` 数据是动态的, 要想获得有效的数据,  请保证应用处于较大的负载 (比如正在生成中运行的服务, 或者通过其他工具模拟访问压力时) 。


* 例子:

```go
package main

import (
	"flag"
	"fmt"
	"os"
	"runtime/pprof"
	"time"
)

// 一段有问题的代码
func logicCode() {
	var c chan int
	for {
		select {
		// 因为c 未初始化 一直在此分支中等待 值
		case v := <-c:
			fmt.Printf("recv from chan, value:%v\n", v)
		default:
			// 未做任何处理
		}
	}
}

func main() {
	var isCPUPprof bool
	var isMemPprof bool

	flag.BoolVar(&isCPUPprof, "cpu", false, "turn cpu pprof on")
	flag.BoolVar(&isMemPprof, "mem", false, "turn mem pprof on")
	flag.Parse()

	if isCPUPprof {
		file, err := os.Create("./cpu.pprof")
		if err != nil {
			fmt.Printf("create cpu pprof failed, err:%v\n", err)
			return
		}
		_ = pprof.StartCPUProfile(file)
		defer pprof.StopCPUProfile()
	}
	for i := 0; i < 8; i++ {
		go logicCode()
	}
	time.Sleep(20 * time.Second)
	if isMemPprof {
		file, err := os.Create("./mem.pprof")
		if err != nil {
			fmt.Printf("create mem pprof failed, err:%v\n", err)
			return
		}
		_ = pprof.WriteHeapProfile(file)
		_ = file.Close()
	}
}

```

```shell
# go run main.go -cpu

```


* 分析生成的报告

```shell
# go tool pprof ./cpu.pprof

```

* 进入交互模式


```shell
Type: cpu
Time: Dec 25, 2019 at 4:00pm (CST)
Duration: 20.14s, Total samples = 1.85mins (550.58%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```


* `top` 命令 查看占用资源排行

```shell
(pprof) top 4
Showing nodes accounting for 110.86s, 100% of 110.89s total
Dropped 9 nodes (cum <= 0.55s)
      flat  flat%   sum%        cum   cum%
    45.01s 40.59% 40.59%     86.34s 77.86%  runtime.selectnbrecv
    30.19s 27.23% 67.81%     33.47s 30.18%  runtime.chanrecv
    24.52s 22.11% 89.93%    110.86s   100%  main.logicCode
    11.14s 10.05%   100%     11.14s 10.05%  runtime.newstack
```

* 选项解释:

  * `flat`: 当前函数占用CPU的耗时

  * `flati%`: 当前函数占用CPU的耗时百分比

  * `sun%`: 函数占用CPU的耗时累计百分比

  * `cum`: 当前函数加上调用当前函数的函数占用CPU的总耗时

  * `cum%`: 当前函数加上调用当前函数的函数占用CPU的总耗时百分比

  * 最后一列: 函数名称


* `list` 命令 分析具体程序问题


```shell
(pprof) list main.logicCode
Total: 1.85mins
ROUTINE ======================== main.logicCode in main.go
    24.52s   1.85mins (flat, cum)   100% of Total
         .          .     12:func logicCode() {
         .          .     13:   var c chan int
         .          .     14:   for {
         .          .     15:           select {
         .          .     16:           // 因为c 未初始化 一直在此分支中等待 值
    24.52s   1.85mins     17:           case v := <-c:
         .          .     18:                   fmt.Printf("recv from chan, value:%v\n", v)
         .          .     19:           default:
         .          .     20:                   // 未做任何处理
         .          .     21:           }
         .          .     22:   }

```

* `24.52s   1.85mins     17:           case v := <-c:`  这一段标注了具体的耗时,说明有问题





