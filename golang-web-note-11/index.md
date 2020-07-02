# Context


# Go Web 编程


## Context

* Go 语言的 Context 既, `上下文`。 是Golang 语言标准库 `golang.org/x/net/context`。

  * golang 的 Context包, 是专门用来简化对于处理单个请求的多个`goroutine`之间与请求域的数据、取消信号、截止时间等相关操作, 这些操作可能涉及多个`API`调用。

  * 比如有一个网络请求`Request`, 每个`Request`都需要开启一个`goroutine`做一些事情, 这些`goroutine`又会开启其他的`goroutine`。这样的话, 我们就可以通过`Context`, 来跟踪这些`goroutine`, 并且可以通过`Context`来控制他们, 这就是Go语言为我们提供的`Context`。

  * 在Go服务器程序中, 每个请求都会有一个`goroutine`去处理。然而, 处理程序往往还需要创建额外的`goroutine`去访问后端资源, 比如数据库、RPC服务等。由于这些`goroutine`都是在处理同一个请求, 所以它们往往需要访问一些共享的资源, 比如用户身份信息、认证token、请求截止时间等。而且如果请求超时或者被取消后, 所有的`goroutine`都应该马上退出并且释放相关的资源。这种情况也需要用`Context`来为我们取消掉所有`goroutine`。


* `context.Context` 实例是一个接口, 该接口定义了四个需要实现的方法。

```go

type Context interface {
    // 返回一个设置结束的时间，和OK
    Deadline() (deadline time.Time, ok bool)
    // Done 是一个只读的 channel
    Done() <-chan struct{}
    // 错误
    Err() error
    // 接收一个 key,返回一个 value
    Value(key interface{}) interface{}
}
```

* 方法解释:

  * `Deadline` 方法 - 需要返回当前`Context`被取消的时间, 也就是完成工作的截止时间（deadline）。 

  * `Done`方法 - 需要返回一个`Channel`, 这个`Channel`会在当前工作完成或者上下文被取消之后关闭, 多次调用`Done`方法会返回同一个`Channel`。

  * `Err`方法 - 返回当前`Context`结束的原因, 它只会在`Done`返回的`Channel`被关闭时才会返回非空的值。
  
    * 如果当前`Context`被取消就会返回`Canceled`错误。

    * 如果当前`Context`超时就会返回`DeadlineExceeded`错误。

  * `Value`方法 - 会从`Context`中返回键对应的值, 对于同一个上下文来说, 多次调用 `Value` 并传入相同的`Key`会返回相同的结果，该方法仅用于传递跨API和进程间跟请求域的数据。


### `Background()`和`TODO()`

* Go内置两个函数: `Background()`和`TODO()`, 这两个函数分别返回一个实现了`Context`接口的`background()`和`todo()`。我们代码中最开始都是以这两个内置的上下文对象作为最顶层的`partent context`, 衍生出更多的子上下文对象。

  * `Background()` - 主要用于main函数、初始化以及测试代码中, 作为`Context`这个树结构的最顶层的`Context`, 也就是根`Context`。

  * `TODO()` - 它目前还不知道具体的使用场景, 如果我们不知道该使用什么`Context`的时候, 可以使用这个。

  * `background`和`todo` - 本质上都是`emptyCtx`结构体类型, 是一个不可取消, 没有设置截止时间, 没有携带任何值的`Context`。


### With 系列函数

* `context`包中定义了四个With系列函数。


* `WithCancel`

```go
// WithCancel 需在Main中使用,并且需要传根Context。( Background()或TODO() )
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// 返回一个 ctx Context 和 一个 cancel函数
```

* `WithCancel` - 返回带有新`Done`通道的父节点的副本。当调用返回的`cancel`函数或当关闭父上下文的`Done`通道时, 将关闭返回上下文的`Done`通道, 无论先发生什么情况。



* 例子:

```go
package main

import (
	"context"
	"fmt"
)

// 创建一个 gen 函数, 返回一个 只读 int类型的 channel
func gen(ctx context.Context) <-chan int {
	// 创建并初始化一个 int 类型的通道
	dst := make(chan int)
	n := 1
	// 启动 goroutine
	// 闭包
	go func() {
		// 无限循环
		for {
			select {
			// 当 ctx.Done() 可以取到值的时候
			case <-ctx.Done():
				// return 结束该 goroutine 防止内存泄露
				return
			case dst <- n:
				// n = 1; n = n + 1
				n++
			}
		}
	}()
	// 返回 dst 通道
	return dst
}
func main() {
	// 创建一个 ctx 子 context
	ctx, cancel := context.WithCancel(context.Background())
	// main函数运行完以后执行 cancel。 退出 goroutine 。
	// main函数通过 ctx ( context ) 传递信号。
	defer cancel()
	// 循环 gen 函数 return 回来的 dst 通道
	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			break
		}
	}
}

```


* `WithDeadline`

  *  `func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)` 函数

  * `WithDeadline` - 返回父上下文的副本, 并将`deadline`调整为不迟于 设置的时间 `d`。如果父上下文的`deadline`已经早于 设置的时间 `d`, 则`WithDeadline(parent, d)`在语义上等同于父上下文。当截止日过期时, 当调用返回的`cancel`函数时, 或者当父上下文的`Done`通道关闭时, 返回上下文的`Done`通道将被关闭, 以最先发生的情况为准。


```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 获取当前时间 加 50 毫秒
	d := time.Now().Add(50 * time.Millisecond)

	// context 调用 withDeadline 传入 d
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// main 函数执行完以后,发送关闭指令
        // ctx 会自动过期,但是官方建议还是尽量调用 cancel() 。
	defer cancel()

	select {
	// 1秒 后 会走 这个 case 分支
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	// d = 50毫秒 结束, 会先走这个 case 分支
	case <-ctx.Done():
		// context 会报: context deadline exceeded 错误
		fmt.Println(ctx.Err())
	}
}

```

* 输出:

```shell
context deadline exceeded

```



* `WithTimeout`

  * `func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)` 
  
  * `WithTimeout` - 返回 `WithDeadline(parent, time.Now().Add(timeout))` , 一个时间间隔。

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// context.WithTimeout

// 定义全局的 等待组
var wg sync.WaitGroup

func worker(ctx context.Context) {
LOOP:
	for {
		fmt.Println("db connecting ...")
		// 假设正常连接数据库耗时10毫秒
		time.Sleep(time.Millisecond * 10)
		select {
		// 第一个case 分支 50毫秒后自动调用
		case <-ctx.Done():
			// 跳出这 for 循环到 LOOP 标签
			break LOOP
		default:
		}
	}
	// WithTimeout 超时,或者手动调用了 cancel() 会走到这里。
	fmt.Println("worker done!")
	// 完成 等待组
	wg.Done()
}

func main() {
	// 设置 timeout = 50毫秒 超时。
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	// 增加一个 等待组
	wg.Add(1)
	// 运行 goroutine 执行 worker 函数
	go worker(ctx)

	// 等待 5 秒
	for i := 1; i < 6; i++ {
		fmt.Printf("waiting %d ...\n", i)
		time.Sleep(time.Second * 1)
	}

	// 通知 子 goroutine 结束
	cancel()
	// 等待组 等待 wg.Done()
	wg.Wait()
	fmt.Println("over")
}

```


```shell
waiting 1 ...
db connecting ...
db connecting ...
db connecting ...
db connecting ...
db connecting ...
worker done!
waiting 2 ...
waiting 3 ...
waiting 4 ...
waiting 5 ...
over
```


* `WithValue`

  * `func WithValue(parent Context, key, val interface{}) Context` 

  * `WithValue` - 返回父节点的副本, 其中与`key`关联的值为`val`。

  * `WithValue` - 仅对API和进程间传递请求域的数据使用上下文值, 而不是使用它来传递可选参数给函数。所提供的键必须是可比较的, 并且不应该是`string`类型或任何其他内置类型, 以避免使用上下文在包之间发生冲突。`WithValue`的用户应该为键定义自己的类型。为了避免在分配给`interface{}`时进行分配, 上下文键通常具有具体类型`struct{}`。或者, 导出的上下文关键变量的静态类型应该是指针或接口。


  
```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

// context.WithValue

type TraceCode string

var wg sync.WaitGroup

func worker(ctx context.Context) {
	key1 := TraceCode("TRACE_CODE1")
	key2 := TraceCode("TRACE_CODE2")
	// 在子goroutine中获取trace code 1
	traceCode1, ok := ctx.Value(key1).(string)
	traceCode2, ok := ctx.Value(key2).(int64)
	if !ok {
		fmt.Println("invalid trace code")
	}
LOOP:
	for {
		fmt.Printf("worker, trace code 1:%s\n", traceCode1)
		fmt.Printf("worker, trace code 2:%d\n", traceCode2)
		// 假设正常连接数据库耗时10毫秒
		time.Sleep(time.Millisecond * 10)
		select {
		// 50毫秒后自动调用
		case <-ctx.Done():
			break LOOP
		default:
		}
	}
	fmt.Println("worker done!")
	wg.Done()
}

func main() {
	// 设置一个50毫秒的超时
	ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond*50)
	// 在系统的入口中设置trace code传递给后续启动的 goroutine
	ctx = context.WithValue(ctx, TraceCode("TRACE_CODE1"), "12512312234")
	ctx = context.WithValue(ctx, TraceCode("TRACE_CODE2"), int64(646464646464646464))

	wg.Add(1)
	go worker(ctx)
	time.Sleep(time.Second * 5)
	// 通知子goroutine结束
	cancel()
	wg.Wait()
	fmt.Println("over")
}

```



* 输出:

```shell
worker, trace code 1:12512312234
worker, trace code 2:646464646464646464
worker, trace code 1:12512312234
worker, trace code 2:646464646464646464
worker, trace code 1:12512312234
worker, trace code 2:646464646464646464
worker, trace code 1:12512312234
worker, trace code 2:646464646464646464
worker, trace code 1:12512312234
worker, trace code 2:646464646464646464
worker done!
over

```



* `context` 注意事项

  * 推荐以参数的方式显示传递`Context`

  * 以`Context`作为参数的函数方法, 应该把`Context`作为第一个参数。

  * 给一个函数方法传递`Context`的时候, 不要传递`nil`, 如果不知道传递什么, 就使用`context.TODO()`。

  * `Context`的`Value`相关方法应该传递请求域的必要数据, 不应该用于传递可选参数。

  * `Context`是线程安全的, 可以放心的在多个`goroutine`中传递。




