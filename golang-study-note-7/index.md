# Go学习笔记


# Go语言基础


## 单元测试

* 单元测试是指 开发完一个模块 之后自己进行测试.

* 单元测试 很重要, TDD: 测试驱动开发.

* Go语言内置 `testing` 包进行单元测试, 所有以 `*_test.go`为后缀名称的文件, 都可通过 `go test` 命令, 自动执行测试函数, 不会被`go build` 编译.

|类型|格式|作用|
|-|-|-|
|测试函数|函数名前缀为Test|测试程序的一些逻辑行为是否正确|
|基准函数|函数名前缀为Benchmark|测试函数的性能|
|示例函数|函数名前缀为Example|为文档提供示例文档|


例子1

```go
// sum.go

//SubAdd 相加
func SubAdd(a, b int) (sub int) {
	sub = a + b
	return
}

// SubMin 相减
func SubMin(a, b int) (sub int) {
	sub = a - b
	return
}

```

```go

// sum_test.go
func TestSubAdd(t *testing.T) {
	a := 10
	b := 20
	c := a + b
	result := SubAdd(a, b)
	if result != c {
		t.Fatalf("期望得到: %v, 实际得到: %v\n", c, result)
	}
}

func TestSubMin(t *testing.T) {
	a := 10
	b := 20
	c := a - b
	result := SubMin(a, b)
	if result != c {
		t.Fatalf("期望得到: %v, 实际得到: %v\n", c, result)
	}
}

// 执行 go test -v 

/* 输出如下:

=== RUN   TestSubAdd
--- PASS: TestSubAdd (0.00s)
=== RUN   TestSubMin
--- PASS: TestSubMin (0.00s)
PASS
ok      github.com/jicki/testing02       0.004s

*/
```



### 测试组

* 将多个测试用例 在一个函数中完成, 就是 测试组


```go
// sum.go

// Sub 计算
func Sub(a, b int, char string) (ret int) {
	switch char {
	case "+":
		ret = a + b
	case "-":
		ret = a - b
	case "*":
		ret = a * b
	case "/":
		ret = a / b
	}
	return
}
```

```go
// sum_test.go

func TestGroup(t *testing.T) {
	// 定义一个 存放测试数据 的结构体
	type test struct {
		a    int
		b    int
		c    int
		char string
	}

	// 创建一个存放所有测试用例的 map
	var tests = map[string]test{
		"Add":   test{10, 20, 30, "+"},
		"Min":   test{10, 20, -10, "-"},
		"Multi": test{10, 20, 200, "*"},
		"Step":  test{10, 20, 0, "/"},
	}
	for name, tc := range tests {

        // 调用 t.Run  子测试方法 
        // 可使用 go test -run=Sub/Add -v 这种方式调用单个测试
		t.Run(name, func(t *testing.T) {
			ret := Sub(tc.a, tc.b, tc.char)
			if ret != tc.c {
				t.Errorf("期望的结果为: %#v, 实际结果为: %#v", tc.c, ret)
			}
		})
	}
}


/*
运行 go test -v 
输出:
=== RUN   TestGroup
=== RUN   TestGroup/Add
=== RUN   TestGroup/Min
=== RUN   TestGroup/Multi
=== RUN   TestGroup/Step
--- PASS: TestGroup (0.00s)
    --- PASS: TestGroup/Add (0.00s)
    --- PASS: TestGroup/Min (0.00s)
    --- PASS: TestGroup/Multi (0.00s)
    --- PASS: TestGroup/Step (0.00s)
PASS
ok      github.com/jicki/testing03       0.005s

*/

```


### 测试覆盖率

* 测试覆盖率是指 代码被测试套件覆盖的百分比, 通常使用的都是语句覆盖率, 测试中至少被运行一次的代码占总代码的比例.

* Go 语言中使用 `go test -cover` 来查看测试覆盖率



### 基准测试 (Benchmark)

* 基准测试就是 性能测试的一种.
* 默认情况下每次最少执行 `1s`,如果不足 `1s` 会重复执行,b.N的值会按1,2,5,10,20,50..增加


例子
```go

func BenchmarkSub(b *testing.B) {
	b.Log("基准测试 !")
	// b.N 是内置方法
	for i := 0; i < b.N; i++ {
		Sub(10, 20, "*")
	}
}


/* 执行 go test -bech=Sub   (正则匹配名称)
输出:

goos: darwin
goarch: amd64
pkg: github.com/jicki/testing03
// 8 是指 8核(可用-cpu 指定)   300000000 是指 执行次数
// 4.16 ns/op 是指每次运行消耗的时间(ms)
// 如果想查看具体的内存信息,可使用 -benchmem 参数查看运行
BenchmarkSub-8          300000000                4.17 ns/op
--- BENCH: BenchmarkSub-8
    sum_test.go:32: 基准测试 !
    sum_test.go:32: 基准测试 !
    sum_test.go:32: 基准测试 !
    sum_test.go:32: 基准测试 !
    sum_test.go:32: 基准测试 !
    sum_test.go:32: 基准测试 !
PASS
ok      github.com/jicki/testing03       1.688s
*/


```


**性能比较函数**

* 在基准测试的前提下,可以对测试进行比较


例子

```go
//fib.go

//Fib 斐波那契数列
func Fib(n int) int {
	if n < 2 {
		return n
	}
	// 递归
	return Fib(n-1) + Fib(n-2)
}
```

```go
// 基准测试
func BenchmarkFib(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Fib(2)
	}
}

// 基准测试 - 性能比较函数 - (内部调用函数)
func benchmarkFib(b *testing.B, n int) {
	for i := 0; i < b.N; i++ {
		Fib(n)
	}
}

// 调用上面的内部函数 benchmarkFib n=2
func BenchmarkFib2(b *testing.B) {
	for i := 0; i < b.N; i++ {
		benchmarkFib(b, 2)
	}
}

// 调用上面的内部函数 benchmarkFib n=20
func BenchmarkFib20(b *testing.B) {
	for i := 0; i < b.N; i++ {
		benchmarkFib(b, 20)
	}
}


/* 执行 go test -bench=.  (.表示执行当前的所有测试函数)
输出:

goos: darwin
goarch: amd64
pkg: github.com/jicki/testing04
BenchmarkFib-8          300000000                5.10 ns/op
BenchmarkFib2-8            30000            157001 ns/op
BenchmarkFib20-8             300          13065579 ns/op
PASS
ok      github.com/jicki/testing04       11.643s

*/
```


**重置时间**

* 函数内某些操作是不需要计算在函数性能内的,比如连接数据库,连接外部api等.
* 函数内可使用 `b.ResetTimer()` 进行重置时间.


```go

func BenchmarkJob(b *testing.B) {
	//1. 连接数据库
	//2. 调用Api
	b.ResetTimer() //重置时间
	// 测试的函数内容
	for i := 0; i < b.N; i++{
		Job()
	}
}

```


**并行测试**

* 开启并行测试，需要执行 `b.RunParallel(func(pb *testing.PB)` 方法，默认会以逻辑CPU个数来进行并行测试.


例子

```go
// sub.go

// SubAdd 加法
func SubAdd(a, b int) (sub int) {
	return a + b
}
```

```go
// sub_test.go
// 基准测试
func BenchmarkSub(b *testing.B) {
	for i := 0; i < b.N; i++ {
		SubAdd(10, 20)
	}
}

// 并行测试

//函数名 约定包含 Parallel 表示 并行测试
//当函数包含很多 groutine 时 使用并行测试
func BenchmarkSubParallel(b *testing.B) {
	// 并行测试的函数 方法
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			SubAdd(10, 20)
		}
	})
}

/* 执行 go test -bench=.

输出:

goos: darwin
goarch: amd64
pkg: github.com/jicki/testing05
BenchmarkSub-8                  2000000000               0.57 ns/op
BenchmarkSubParallel-8          2000000000               0.24 ns/op
PASS
ok      github.com/jicki/testing05       1.706s
*/

```


### Setup与TearDown

* Setup 意指 在测试之前需要进行设置,如连接数据库等.
* TearDown 意指 在测试之后需要进行卸载


**TestMain**

* 在测试文件中调用 `func TestMain(m *testing.M)`  那么程序在生成测试之前会先调用 `TestMain(m)`,然后在执行具体测试.
* `TestMain` 运行在 `goroutine` 中, 调用 `m.Run` 前后做 `setup`和`teardown`, 退出测试时使用 `m.Run` 的返回值做为参数调用`os.Exit`.

例子:

```go

// TestMain
func TestMain(m *testing.M) {
	fmt.Println("测试之前的操作(Setup)")
	//m.Run() 执行以上的测试,返回值为测试结果
	ret := m.Run()
	fmt.Println("测试结束后执行的操作(teardown)")
	// 调用系统退出,把测试结果传进去
	os.Exit(ret)
}


/* 执行 go test -bench=.
输出:

测试之前的操作(Setup)
goos: darwin
goarch: amd64
pkg: github.com/jicki/testing06
BenchmarkSub-8                  2000000000               0.57 ns/op
BenchmarkSubParallel-8          2000000000               0.23 ns/op
PASS
测试结束后执行的操作(teardown)
ok      github.com/jicki/testing06       1.689s

*/
```

**子测试(Setup/Teardown)**


例子:

```go
func Sub(a, b int, char string) (ret int) {
	switch char {
	case "+":
		ret = a + b
	case "-":
		ret = a - b
	case "*":
		ret = a * b
	case "/":
		ret = a / b
	}
	return
}
```

```go
// sum_test.go

func TestGroup(t *testing.T) {
	// 定义一个 存放测试数据 的结构体
	type test struct {
		a    int
		b    int
		c    int
		char string
	}

	// 创建一个存放所有测试用例的 map
	var tests = map[string]test{
		"Add":   test{10, 20, 30, "+"},
		"Min":   test{10, 20, -10, "-"},
		"Multi": test{10, 20, 200, "*"},
		"Step":  test{10, 20, 0, "/"},
	}
	for name, tc := range tests {
		t.Run(name, func(t *testing.T) {
			// 在t.Run 这个子测试中,加入 Setup 与 Teardown
			t.Log("t.Run 子测试之前执行 Setup")
			defer func() {
				t.Log("t.Run 子测试之后执行 Teardown")
			}()
			ret := Sub(tc.a, tc.b, tc.char)
			t.Log("子测试实际执行过程!!")
			if ret != tc.c {
				t.Errorf("期望的结果为: %#v, 实际结果为: %#v", tc.c, ret)
			}
		})
	}
}


/*
输出:
=== RUN   TestGroup
=== RUN   TestGroup/Multi
=== RUN   TestGroup/Step
=== RUN   TestGroup/Add
=== RUN   TestGroup/Min
--- PASS: TestGroup (0.00s)
    --- PASS: TestGroup/Multi (0.00s)
        sum_test.go:24: t.Run 子测试之前执行 Setup
        sum_test.go:29: 子测试实际执行过程!!
        sum_test.go:26: t.Run 子测试之后执行 Teardown
    --- PASS: TestGroup/Step (0.00s)
        sum_test.go:24: t.Run 子测试之前执行 Setup
        sum_test.go:29: 子测试实际执行过程!!
        sum_test.go:26: t.Run 子测试之后执行 Teardown
    --- PASS: TestGroup/Add (0.00s)
        sum_test.go:24: t.Run 子测试之前执行 Setup
        sum_test.go:29: 子测试实际执行过程!!
        sum_test.go:26: t.Run 子测试之后执行 Teardown
    --- PASS: TestGroup/Min (0.00s)
        sum_test.go:24: t.Run 子测试之前执行 Setup
        sum_test.go:29: 子测试实际执行过程!!
        sum_test.go:26: t.Run 子测试之后执行 Teardown
PASS
ok      github.com/jicki/testing07       0.005s

*/

```


### 示例函数

* 示例函数 以 `Example` 为前缀. 它既没有参数也没有返回值.

```go
// sub.go

// SubAdd 加法
func SubAdd(a, b int) (sub int) {
	return a + b
}
```

```go
// sub_test.go

// 示例函数 
// 必须写注释  OutPut: 然后加上结果
func ExampleSubAdd() {
	fmt.Println(SubAdd(10, 20))
	// OutPut: 30
}


/*
输出:
=== RUN   ExampleSubAdd
--- PASS: ExampleSubAdd (0.00s)
PASS
ok      github.com/jicki/testing08       0.005s
*/
```


## Go网络编程


### OSI七层模型

* 互联网协议

```
# OSI 七层协议

				   -->  [应用层]
[应用层]           --> [应用层]    -->  [表示层]
				   -->  [会话层]

[传输层]           --> [传输层]    -->  [传输层]


[网络层]           --> [网络层]    -->  [网络层]

		   --> [数据链路层] --> [数据链路层]	
[网络接口层]              
		   --> [物理层]    --> [物理层]
```



### TCP/IP 协议


**TCP协议**

* TCP(Transmission Control Protocol) 既传输控制协议/网络协议, 是一种面向连接(连接导向)的、可靠的、基于字节流的传输(Transport layer) 通信协议，因为是面向连接的协议，数据像水流一样传输，所以会存在黏包问题。



**IP协议**

* IP(Internet Protocol) 因特网协议是为计算机网络相互连接进行通信而设计的协议。在因特网中，它规定了计算机在因特网中进行通信应该遵守的规则。

* IP协议中还有一个非常重要的内容，给因特网上的每台计算机和设备都规定了一种地址，叫做"IP地址"。


### Socket

* `Socket` 是 `BSD UNIX` 的进程通讯机制, 通常也称为 "套接字" , 用于描述IP地址和端口，是一个通讯链的句柄.
* `Socket` 可以理解为 `TCP/IP` 网络的API，它定义了很多函数或例程，程序员可以用它们来开发`TCP/IP`网络上的应用程序.

* `Socket` 是应用层与 `TCP/IP` 协议族通信的中间软件抽象层,在设计模式中，`Socket` 其实就是一个门面模式,它把复杂的 `TCP/IP` 协议族隐藏在 `Socket` 后面,对用户来说只需要调用 `Socket` 规定的相关函数，让 `Socket` 去组织符合指定的协议数据然后进行通信。


```
# sokcet 抽象层 位于 应用层 与 传输层之间

    [应用层]
       |
[Socket 抽象层]
       |
    [传输层]
       |
    [网络层]
       |
   [数据链路层]
       |
    [物理层]
```

### TCP服务端/客户端


**服务端**

* Go语言可通过 `net` 包实现 `TCP`服务端



**server**

```go
// 处理连接的函数
func process(conn net.Conn) {
	//从连接中接收数据

	// 关闭客户端连接
	defer conn.Close()
	// 创建一个 切片
	buf := make([]byte, 1024)
	// n 表示读了多少数据(byte)
	n, err := conn.Read(buf)
	if err != nil {
		fmt.Println("接收客户端数据失败,err:", err)
	}
	// buf 为字节切片，需要转换为 string
	fmt.Println("客户端发来的消息: ", string(buf[:n]))
}

func main() {
	listener, err := net.Listen("tcp", "127.0.0.1:8888")
	if err != nil {
		fmt.Println("Listen 失败!", err)
		return
	}
	fmt.Println("Listener: ", listener.Addr())
	// 关闭 服务端连接 释放服务端绑定端口
	defer listener.Close()
	//创建一个 for 循环，一直接收信息
	for {
		// 如果没有连接，会一直等待客户端连接.
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("客户端连接失败,err:", err)
			continue
		}
		// 客户端连接成功 使用 goroutine 创建 连接
		go process(conn)
	}
}

```


**Client**

```go
//tcp 客户端
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:8888")
	if err != nil {
		fmt.Println("服务端连接失败,err:", err)
		return
	}
	// 关闭连接
	defer conn.Close()
	// 使用 bufio 获取 终端输入(os.Stdin)
	for {
		reader := bufio.NewReader(os.Stdin)
		// 读取终端输入的内容,以换行(\n)为结束
		input, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("获取终端输入失败 err:", err)
			return
		}
		data := []byte(input)
		_, err = conn.Write(data)
		if err != nil {
			fmt.Println("发送消息失败 err:", err)
			return
		}
	}
}

```



### TCP黏包问题

**Server**

```go

func process(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	for {
                // 调用模块proto.Decode 函数
		msg, err := proto.Decode(reader)
		if err == io.EOF {
			return
		}
		if err != nil {
			fmt.Println("消息解码失败 err: ", err)
		}
		fmt.Println("收到客户端发送的消息: ", msg)
	}
}

func main() {
	listen, err := net.Listen("tcp", "127.0.0.1:8888")
	if err != nil {
		fmt.Println("启动服务端失败 err: ", err)
		return
	}
	for {
		conn, err := listen.Accept()
		if err != nil {
			fmt.Println("客户端连接失败, err", err)
			continue
		}
		go process(conn)
	}
}

```


**Client**

```go

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:8888")
	if err != nil {
		fmt.Println("连接服务端失败, err ", err)
		return
	}
	defer conn.Close()
	for i := 0; i < 20; i++ {
		msg := "Hello How are you ?"
                // 调用模块 proto.Encode
		pkg, err := proto.Encode(msg)
		if err != nil {
			fmt.Println("Encode Error:", err)
			return
		}
		conn.Write(pkg)
	}
}

```


**模块**

```go
//Encode 消息编码
func Encode(message string) ([]byte, error) {
	// length 将消息长度,转换成int32类型(int32 占4字节)
	var length = int32(len(message))
	// 创建一个 字节类型的 缓冲区
	var pkg = new(bytes.Buffer)

	// 写入信息头 pkg
	// binary.LittleEndian 内存 小端.
	// 小端: 内存读写的顺序,按照从左到右的顺序是小端.
	// 大端: 内存读写的顺序,按照从右到左的顺序是大端.
	if err := binary.Write(pkg, binary.LittleEndian, length); err != nil {
		return nil, err
	}

	// 写入消息内容
	if err := binary.Write(pkg, binary.LittleEndian, []byte(message)); err != nil {
		return nil, err
	}
	return pkg.Bytes(), nil
}

// Decode 解码消息
func Decode(reader *bufio.Reader) (string, error) {
	// 读取消息头
	// 读取消息前4个字节
	lengthByte, _ := reader.Peek(4)
	// 缓冲区的内容
	lengthBuff := bytes.NewBuffer(lengthByte)
	var length int32
	// 读取消息
	err := binary.Read(lengthBuff, binary.LittleEndian, &length)
	if err != nil {
		return "", err
	}
	// 返回(缓冲区内)消息头中记录真正消息的(大小)字节数
	if int32(reader.Buffered()) < length+4 {
		return "", err
	}

	// 读取真正的消息内容
	// 创建还有个切片,用于存放消息内容, 长度为头部4+length
	pack := make([]byte, int(4+length))
	_, err = reader.Read(pack)
	if err != nil {
		return "", err
	}
	// 返回切数据中从第四位开始后面的数据
	return string(pack[4:]), nil
}

```




### UDP服务端/客户端

* UDP协议(User Datagram Protocol) 用户数据报协议, 是 OSI 参考模型中一种 无连接 的传输层协议, 不需要建立连接就可以直接进行数据发送和接收, 属于不可靠的、无时序的通信, UDP协议的实时性比较好,通常用于视频直播相关领域.


**Server服务端**

```go

func main() {
	listen, err := net.ListenUDP("udp", &net.UDPAddr{
		IP:   net.ParseIP("127.0.0.1"),
		Port: 9999,
	})
	if err != nil {
		fmt.Println("Listen 失败! err:", err)
		return
	}
	defer listen.Close()
	// 循环收发数据
	data := make([]byte, 1024)
	for {
		n, addr, err := listen.ReadFromUDP(data)
		if err != nil {
			fmt.Println("接收消息失败! err :", err)
			return
		}
		fmt.Printf("接收到来自 %v 的消息 %v\n", addr, string(data[:n]))
		// 回复消息
		n, err = listen.WriteToUDP([]byte("收到收到!"), addr)
		if err != nil {
			fmt.Println("回复消息失败! err: ", err)
			return
		}
	}
}

```

**Client客户端**


```go
//udp client
func main() {
	conn, err := net.Dial("udp", "127.0.0.1:9999")
	if err != nil {
		fmt.Println("连接服务端失败! err: ", err)
	}
	defer conn.Close()
	// 发送消息
	n, err := conn.Write([]byte("哈喽"))
	if err != nil {
		fmt.Println("发送消息失败 err: ", err)
		return
	}
	// 接收消息
	buf := make([]byte, 1024)
	n, err = conn.Read(buf)
	if err != nil {
		fmt.Println("接收信息失败 err: ", err)
		return
	}
	fmt.Println("收到信息 :", string(buf[:n]))
}
```


### Http

* HTTP 超文本传输协议(HTTP , HyperText Transfer Protocol) 是互联网上应用最为广泛的一种网络传输协议, 所有的 WWW 文件都必须遵守这个标准.
* 最初设计HTTP是为了提供一种发布和接收HTML页面的方法.


**HTTP数据传输图解**


```
                    [HTTP数据]                  [应用层] - HTTP、FTP、SMTP
          		|  
   		[TCP部首(HTTP数据)]		[传输层] - TCP、UDP
		 	|
	[IP首部(TCP部首)(HTTP数据)]		[网络层] - IP、ARP、路由器
		 	|
[以太网部首(IP部首)(TCP首部)(HTTP数据)]		[数据链路层] - 以太网、网桥

					 物理层

```


**模拟httpClient**

```go
func main() {
	conn, err := net.Dial("tcp", "baidu.com:80")
	if err != nil {
		fmt.Println("打开网站失败,err: ", err)
	}
	defer conn.Close()
	// 发送数据到网站
	//fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
	conn.Write([]byte("GET / HTTP/1.0\r\n\r\n"))
	//接收数据
	buf := make([]byte, 1024)
	for {
		n, err := conn.Read(buf)
		if err == io.EOF {
			fmt.Printf(string(buf[:n]))
			return
		}
		if err != nil {
			fmt.Println("接收数据失败 err: ", err)
			return
		}
		fmt.Printf(string(buf[:n]))
	}
}
```


**HttpServer**

* 使用Go语言中的 `http` 包

```go
func process(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "<h1>My Http Server!</h1>")
}

// http Server
func main() {
	// 注册路由
	http.HandleFunc("/", process)
	// 创建连接
	err := http.ListenAndServe(":9999", nil)
	if err != nil {
		fmt.Println("Listen Error:", err)
		return
	}
}
```

