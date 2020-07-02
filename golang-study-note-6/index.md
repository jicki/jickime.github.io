# Go学习笔记


# Go语言基础


## 并发 

* 并发与并行的区别:
    1. 并行 - 是指在一个处理器上“同时”处理多个任务。
    2. 并发 - 是指在多个处理器上同时处理多个任务。




## 进程、线程、协程

* 进程: 一个程序启动后就是一个进程。 (进程是系统资源分配的最小单位.)
  
* 线程: 线程就是运行在进程上下文中的逻辑流。 (线程是操作系统能够进行运算调度的最小单位.)

* 协程: 协程又称微线程和纤程, 协没有线程的上下文切换消耗。协程的调度切换是用户(程序员)手动切换的,因此更加灵活,因此又叫用户空间线程。

## goroutine

* Go 语言中创建 `goroutine` 使用 关键字 `go` 就可以为函数创建一个 `goroutine` 。 
* 一个函数可以创建多个 `goroutine` 。
* 一个 `goroutine` 只能对应一个函数。
* `goroutine` 调度是随机、无序的。



`sync.WaitGroup` 配合 `goroutine` 使用

`sync.WaitGroup` 包含三个方法 `Add(i)`  `Done()`  `Wait()` 


1. 例子1

```go
package main

import (
	"fmt"
	"sync"
)

// 定义一个锁等待
var wg sync.WaitGroup

func say() {
	fmt.Println("This goroutine Func !")
	// 执行完毕 wg.Add(n)  每执行完毕都 n-1
	wg.Done()
}

func main() {
	// 使用关键字 go 启动一个 goroutine
	// 添加锁等待, (1) 数字为多少个 goroutine
	wg.Add(1)
	go say()
	fmt.Println("This Main Func !")
	// 阻塞, 等待 goroutine 运行完.
	wg.Wait()
}
```


### goroutine 与 线程

**可增长的栈**

* OS线程(操作系统线程)一般都有固定的栈内存(通常2MB), 一个 `goroutine` 的栈在其生命周期开始时占用很小的内存(一般为2KB), 
`goroutine` 的栈并不是固定的, 它可以按需增加或缩小, `goroutine` 栈的大小限制最大可以达到1GB。


**goroutine调度**

* OS线程是由OS内核来调度, `goroutine` 则是由 Go运行 `runtime` 自己的调度器调度的, 
`goroutine`调度器使用一个称为 m:n 调度技术(复用/调度 m 个 goroutine 到 n 个OS线程)。`goroutine` 调度不需要切换到OS内核环境,
所以调度一个 `goroutine` 比调度一个线程成本低很多。 (m:n  m 是指 goroutine 数量 , n 是指 线程数量。)



**GOMAXPROCS**

* Go运行时的调度器使用 `GOMAXPROCS` 参数来确定需要使用多少个OS线程来同时执行Go代码。默认是机器CPU总核数。

* Go语言可以通过 `runtime.GOMAXPROCS()` 函数设置当前程序并发时占用的CPU逻辑核心数。

1. 例子1

```go
package main

import (
	"fmt"
	"sync"
)

var wg sync.WaitGroup

func a() {
	defer wg.Done()
	for i := 1; i < 10; i++ {
		fmt.Printf("Func A 运行 %d 次\n", i)
	}
}

func b() {
	defer wg.Done()
	for i := 1; i < 10; i++ {
		fmt.Printf("Func B 运行 %d 次\n", i)

	}
}

func main() {
	//runtime.GOMAXPROCS(1) // 设置程序运行占用 多少个 逻辑核数
	wg.Add(10)
	for i := 1; i < 10; i++ {
		go a()
		go b()
	}
	wg.Wait()
}

```


**OS与goroutine的关系**

* 一个操作系统线程对应用户态多个 `goroutine`。 
* Go 程序可以同时使用多个操作系统线程。
* `goroutine` 与 OS 线程 是多对多的关系,既 m:n 。



**goroutine退出**

* `goroutine` 什么时候退出? `goroutine` 是在 `goroutine` 启动所启动的那个 `函数` 退出的时候 就会退出.



## channel

* Go语言的并发模型是CSP, 提倡通过 `通信共享内存` 而不是通过 共享内存 而实现通信。
  `goroutine` 是Go程序的并发执行体, `channel` 就是它们之间的连接。channel 是可以
  让一个 `goroutine` 发送特定值到另一个 `goroutine` 的通信机制。
* Go语言中的 通信(channel) 是一种特殊的类型。通道像一个 传送带或者队列, 总是遵循先入先出(First in First out)
  的规则, 保证收发数据的顺序。每一个通道都是一个具体类型的导管, 也就是声明 `channel` 的时候需要为其指定 元素类型。


**声明channel**

```go
func main() {
	// channl 的定义
	// channel 是引用类型 (引用类型需要 初始化, 未初始化值 nil)
	// var 变量名(ch1) chan(变量类型channel) 
	// int(channel传递数据的类型)
	var ch1 chan int

	fmt.Printf("未初始化 ch1 = %v\n", ch1)
	// 输出 未初始化 ch1 = <nil>

	// var 变量名(ch2) chan(变量类型channel)
	// string(channel传递数据的类型)
	// 9 为容量,channel 中可接收多少个数据
	var ch2 chan string
	ch2 = make(chan string, 9)
	fmt.Printf("初始化 ch2 = %v\n", ch2)
	// 输出 初始化 ch2 = 0xc0000a2060

	// 直接定义以及初始化
	ch3 := make(chan int, 10)

	// 操作 channel , 发送, 接收, 关闭
	// 发送与接收 使用符号 <-
	ch3 <- 10 // 将 10 发送到 ch3 中
	// <-ch3        //接收值,并直接丢弃接收的值
	ret := <-ch3 //接收值,并保存到ret变量中.

	fmt.Println(ret)
	// 输出 10

	// 关闭管道
	// 1. 关闭的通道可以继续取值,值为传递类型的零值
	// 2. 关闭的通道不允许发送值,会直接 panic
	// 3. 重复关闭已关闭的通道,会直接 panic
	close(ch3)
}
```


**无缓存与有缓存通道**

* 无缓冲的与有缓冲channel有着重大差别：一个是同步的 一个是非同步的

* 无缓冲的  就是一个送信人去你家门口送信 ，你不在家 他不走，你一定要接下信，他才会走。无缓冲保证信能到你手上

* 有缓冲的 就是一个送信人去你家仍到你家的信箱 转身就走 ，除非你的信箱满了 他必须等信箱空下来。有缓冲的 保证 信能进你家的邮箱

```go  
ch1 := make(chan int)        //无缓冲

ch2 := make(chan int,1)      //有缓冲
```

* 例子

```go
// 有缓冲通道 与 无缓冲通道

func recv(ch chan bool) {
	ret := <-ch //接收 channel 数据, 阻塞
	fmt.Println("recv 函数 通道接收数据 :", ret)
}

func main() {
	ch := make(chan bool, 1)
	ch <- false
	go recv(ch)
	ch <- true
	fmt.Println("Main 函数结束")
}

```



**判断通道是否被关闭**

* 通道取值的时候如果通道被关闭,仍然取值, 就会 panic, 所以可以使用
  value, ok := chan 中的OK来判断, 或者使用 for range 来循环取值

```go

// 通道 接收值 的时候判断通道是否关闭
func send(ch chan int) {
	for i := 0; i < 10; i++ {
		ch <- i
	}
	close(ch)
}

func main() {
	ch1 := make(chan int)
	go send(ch1)
	// for 循环 去通道不断的取值
	// 判断 接收值 的时候判断通道是否关闭
	// 方法一: 利用 value 与 ok 判断
	for {
		ret, ok := <-ch1
		if !ok {
			break
		}
		fmt.Println(ret)
	}
	// 方法二: 利用for range 循环取值 (推荐)
	for ret := range ch1 {
		fmt.Println(ret)
	}
}

```



**生产者消费者模型**

```go

func produce(ch chan int) {
	for i := 0; i < 10; i++ {
		ch <- i
		fmt.Println("Send:", i)
	}
	close(ch)
}

func consumer(ch chan int) {
	for v := range ch {
		fmt.Println("Receive:", v)
	}
}

func main() {
	// 无缓冲区,send 一个,接受一个.
	ch := make(chan int)
	// 有缓冲区,send 10个,接收10个.
	//ch := make(chan int, 10)
	go produce(ch)
	go consumer(ch)
	time.Sleep(1 * time.Second)
}

```


**select 多路复用**

* `select` 可以同时响应多个通道的操作。 

```go
// select 语法
select{
	case ch1 <- 1:  // 多通道的操作 发送或者接收值
		...  
	case ch1 <- 2:  // 多通道的操作 发送或者接收值
		...
	case <-ch1:
		...
	case <-ch2:
		...
	default:
		...
}

```



例子1:

```go
func f1(ch chan string) {
	for i := 0; i < 10000; i++ {
		ch <- fmt.Sprintf("f1 Send -> %d", i)
	}
}

func f2(ch chan string) {
	for i := 0; i < 10000; i++ {
		ch <- fmt.Sprintf("f2 Send -> %d", i)
	}
}

func main() {
	ch1 := make(chan string, 100)
	ch2 := make(chan string, 100)
	go f1(ch1)
	go f2(ch2)
	for {
		select {
		case ret := <-ch1:
			fmt.Println(ret)
		case ret := <-ch2:
			fmt.Println(ret)
		default:
			fmt.Println("Null")
			time.Sleep(time.Second * 1)
		}
	}
}
```

例子2:

```go
func main() {
	// 有缓冲区,容量为1
	ch := make(chan int, 1)
	for i := 0; i < 10; i++ {
		select {
		// 1,3,5,7,9 写不进去,因为ch中容量为1
		case ch <- i: //当有1个值,未取时,下个值写不进去
		case ret := <-ch:
			// 输出 0 2 4 6 8
			fmt.Println(ret)
		}
	}
}
```


**单向通道**

* 单向通道 1. 让代码更加的清晰  2. 防止误操作

```go

// 函数参数中包含 chan<- 表示只能 发送
func send(ch chan<- int) {
	ch <- 1
}

// 函数参数中包含 <-chan 表示只能 接收
func receive(ch <-chan int) {
	<-ch
}

```


## 控制与锁

* 多个 `goroutine` 操作同一组数据的时候,会出现数据竞争, 这时候我们就要加锁.


**goroutine 竞争**

```go
var x int64
var wg sync.WaitGroup

func add() {
	for i := 0; i < 5000; i++ {
		// 全局变量 x
		// 每循环一次 x + 1
		x = x + 1
	}
	wg.Done()
}

func main() {
	wg.Add(2)
	// 启动2个 goroutine 去执行 x+1
	// 此时会出现数据竞争
	go add()
	go add()
	wg.Wait()
	// 返回的结果每次都不同
	fmt.Println(x)
}
```


**互斥锁**

* 互斥锁 是一种常用的控制共享资源访问的方法, 它能够保证同一时间只能有一个 `goroutine` 可以访问共享资源.
* Go语言 使用 `sync` 包 `Mutex`  类型来实现 互斥锁.
* 互斥锁能保证同一时间只有一个 `goroutine` 进入临界区, 其他的 `goroutine` 则在等待锁, 当互斥锁释放后, 等待的 `goroutine` 才可以获取锁进入临界区, 多个 `goroutine` 同时等待一个锁时, 唤醒的策略是随机的.


例子1
```go
var x int64
var wg sync.WaitGroup

// 定义一个互斥锁
var lock sync.Mutex

func add() {
	for i := 0; i < 5000; i++ {
		// 加锁
		lock.Lock()
		x = x + 1
		// 解锁
		lock.Unlock()
	}
	wg.Done()
}

func main() {
	wg.Add(2)
	go add()
	go add()
	wg.Wait()
	fmt.Println(x)
}
```

**读写锁**

* 读写锁是一种特殊的自旋锁，它把对共享资源对访问者划分成了读者和写者，读者只对共享资源进行访问，写者则是对共享资源进行写操作。
* 一个读写锁同时只能存在一个写锁但是可以存在多个读锁，但不能同时存在写锁和读锁。
* 如果读写锁当前没有读者，也没有写者，那么写者可以立刻获的读写锁，否则必须自旋，直到没有任何的写锁或者读锁存在。
* 如果读写锁没有写锁，那么读锁可以立马获取，否则必须等待写锁释放。




```go
var (
	x  int64
	wg sync.WaitGroup
	// 互斥锁
	lock sync.Mutex
	// 读写锁
	rwlock sync.RWMutex
)

func read() {
	rwlock.RLock() // 加读锁
	time.Sleep(time.Millisecond)
	rwlock.RUnlock() // 解除 读锁
	wg.Done()
}

func write() {
	rwlock.Lock() // 加写锁
	x = x + 1
	time.Sleep(10 * time.Millisecond)
	rwlock.Unlock() // 解除 写锁
	wg.Done()
}

func main() {
	start := time.Now() // 开始时间

	// 执行 50次 写的操作
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go write()
	}

	// 执行 1000 次的 读操作
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go read()
	}
	wg.Wait()
	end := time.Now() // 结束时间
	fmt.Printf("消耗了 %v 毫秒\n", end.Sub(start))
}
```


**sync.Map**

* Go语言中内置的Map不是并发安全的。

* fatal error: concurrent map writes

* `sync.Map` 不需要(make)初始化,直接可以使用.

* `sync.Map` 可为 map 加锁保证 map 的安全, 而且 `sync.Map` 还内置了 `Store` , `Load` , `LoadOrStore` , `Delete` , `Range` 等方法. 


例子1 (线程不安全的map) 

```go
var m = make(map[string]int)

var wg sync.WaitGroup

func get(key string) int {
	return m[key]
}

func set(key string, value int) {
	m[key] = value
}

func main() {
	// 当 goroutine 超过5个,会报错
	// fatal error: concurrent map writes
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(n int) {
			//将int类型转换成 string类型
			key := strconv.Itoa(n)
			set(key, n)
			fmt.Printf("k = %v,v = %v\n", key, m[key])
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```


例子2

```go
var m = sync.Map{}
var wg sync.WaitGroup

func main() {
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(n int) {
			key := strconv.Itoa(n)
			// sync.Map 用 Store 方法来设置值
			m.Store(key, n)
			value, _ := m.Load(key)
			fmt.Printf("k = %v, v = %v\n", key, value)
			wg.Done()
		}(i)
	}
	wg.Wait()
}

```

**原子操作**

* 在代码中加锁的操作性能会下降. 针对基本数据类型我们可以用 原子操作 来确保并发安全
* 原子操作 是Go语言的方法, 它在用户态 的时候就可以完成, 性能比加锁操作更好.
* Go语言的 原子操作 作为内置的标准库 `sync/atomic` 模块.
* `atomic` 原子操作,只支持 Int, Uint 的数据操作. 


```go
// atomic 原子操作

var (
	x  int64
	l  sync.Mutex     //锁
	wg sync.WaitGroup //等待组
)

// 累加函数
func add() {
	x = x + 1
	wg.Done()
}

// 加锁的累加函数
func mutexAdd() {
	l.Lock()
	x = x + 1
	l.Unlock()
	wg.Done()
}

func atomicAdd() {
	// 给整数 x + 1
	atomic.AddInt64(&x, 1)
	wg.Done()
}

func main() {
	start := time.Now()
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		// 普通的数据累加操作
		// 线程不是安全的,耗时 5.383036ms
		//go add()

		// 加锁版数据累加操作
		// 线程安全的,耗时 5.628079ms
		// go mutexAdd()

		// 原子操作版数据累加
		// 线程安全, 耗时 5.263185ms
		go atomicAdd()
	}

	wg.Wait()
	end := time.Now()
	fmt.Println(x)
	fmt.Println(end.Sub(start))
}
```

