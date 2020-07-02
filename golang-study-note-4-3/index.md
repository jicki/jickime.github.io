# Go学习笔记


# Go语言基础

## 文件操作

* 文件:
  1. 数据源（保存数据数据的地方）
  2. 文件在程序中是以流的形式来操作。
  3. 流是数据在数据源(文件)和程序(内存)之间经历的路径
  4. 输入流: 数据从数据源(文件)到程序(内存)的路径 文件 --> Go程序（内存): 输入流[读文件]
  5. 输出流: 数据从程序(内存)到数据源(文件)的路径 Go程序（内存）--> 文件: 输出流[写文件]

### 打开和关闭文件

* `os.Open()`函数能够打开一个文件, 返回一个`*File`和一个`err`。对得到的文件实例调用`close()`方法能够关闭文件。

```go
// 文件操作
func main() {
	// 调用 os.Open 方法打开 文件
	file, err := os.Open("./temp.txt")
	if err != nil {
		fmt.Println("open file failed!, err:", err)
		return
	}
	// 关闭文件
	defer file.Close()
}

```

### 读取文件

* `file.Read()` 方法  `func (f *File) Read(b []byte) (n int, err error)`  这里传入的变量是 `[]byte` 字节切片, 返回`n`读取行数,与`error`错误。这里判断文件读完了, `n` 返回为0, `error` 返回 `io.EOF` 。

```go
func osRead(f string) {
	file, err := os.Open(f)
	if err != nil {
		fmt.Println("open file failed err:", err)
		return
	}
	defer file.Close()
	// 完整的元素的切片
	var data []byte
	// 每一次读取的元素,临时存储的切片
	var temp = make([]byte, 128)
	// 因为 temp 字节切片 有一定容量,一次读不完,所以可使用 for 循环读取
	for {
		n, err := file.Read(temp)
		// 判断文件是否已经读取完, 用 io.EOF
		if err == io.EOF {
			fmt.Println("文件读完了")
			break
		}
		if err != nil {
			fmt.Println("read file failed err:", err)
			return
		}
		// 利用 append 函数 将临时切片读取到的内容 追加到完整切片中
		data = append(data, temp[:n]...)
	}
	// 字节切片,打印的时候需要转换成 string 类型
	fmt.Println(string(data))
}
```

* `bufio` 读取文件: `bufio`是在`file`的基础上封装了一层API, 支持更多的功能。

```go
func buffIoRead(f string) {
	file, err := os.Open(f)
	if err != nil {
		fmt.Println("open file failed err:", err)
		return
	}
	defer file.Close()
	// 定义一个 缓冲区,用于存放读取到的内容
	reader := bufio.NewReader(file)
	for {
		// 这里 '\n' 注意是字符, 这里意思是读取到什么字符就结束
		line, err := reader.ReadString('\n')
		if err == io.EOF {
			fmt.Println(line)
			fmt.Println("文件读完了")
			break
		}
		if err != nil {
			fmt.Println("read file failed, err:", err)
			return
		}
		fmt.Print(line)
	}
}
```

* `ioutil` 读取整个文件: `io/ioutil`包的`ReadFile`方法能够读取完整的文件, 只需要将文件名作为参数传入。

```go
func iouTileRead(f string) {
	// 使用 iouTil 读取整个文件
	data, err := ioutil.ReadFile(f)
	if err != nil {
		fmt.Println("read file failed err:", err)
		return
	}
	// 返回的是 []byte 也需要 string 转换
	fmt.Println(string(data))
}
```

### 写文件

* `os.OpenFile()`函数能够以指定模式打开文件, 从而实现文件写入相关功能。
* 打开文件格式: `func OpenFile(name string, flag int, perm FileMode) (*File, error) {`
  * `name`: 要打开的文件名。
  * `flag`: 打开文件的模式。
  * `perm`: 文件权限, 一个八进制数。`r（读）04` , `w（写）02` , `x（执行）01` 。
  * 模式有以下几种:

|模式|含义|
|-|-|
|os.O_WRONLY|只写|
|os.O_CREATE|创建文件|
|os.O_RDONLY|只读|
|os.O_RDWR|读写|
|os.O_TRUNC|清空|
|os.O_APPEND|追加|

```go
// 写文件
func fileWrite(f, str string) {
	// 打开文件并配置权限
	file, err := os.OpenFile(f, os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		fmt.Println("open file failed err:", err)
		return
    }
    defer file.Close()
	// 写入文件内容
	_, err = file.WriteString(str)
	if err != nil {
		fmt.Println("write file failed err:", err)
		return
	}
	fmt.Println(str)
}

func main(){
    fileWrite("./xx.txt", "测试写入")
}
```

* `bufio.NewWriter` 写文件。

```go
func buffIoWrite(f, str string) {
	file, err := os.OpenFile(f, os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0644)
	if err != nil {
		fmt.Println("open file failed, err:", err)
		return
	}
	defer file.Close()
    writer := bufio.NewWriter(file)
    // 循环写入多行数据
	for i := 0; i < 10; i++ {
		//将数据先写入缓存
		_, err = writer.WriteString(str)
		if err != nil {
			fmt.Println("WriteString Failed err:", err)
			return
		}
	}
	//因为写入的信息是先存在缓存中,所以需要将缓存中的内容写入文件
	err = writer.Flush()
	if err != nil {
		fmt.Println("Flush Failed err:", err)
		return
	}
}

func main(){
    buffIoWrite("./xx.txt", "buffio 测试写入\n")
}
```

* `ioutil.WriteFile` 写入文件。

```go
func iouTilWrite(f, str string) {
	// 这里写入文件并不能配置 模式,只能一次写入,多次写入会覆盖
	err := ioutil.WriteFile(f, []byte(str), 0666)
	if err != nil {
		fmt.Println("iouTilWrite Failed err:", err)
		return
	}
}

func main(){
    iouTilWrite("./xx.txt", "iouTileWrite 测试写入\n")
}
```

### 实现Copy文件

* 利用文件操作,读取源文件,并存储到 切片中,再写入到 目标文件中。

```go
func copyFile(dstName, srcName string) (n int, err error) {
	// 以只读的方式打开原文件
	src, err := os.Open(srcName)
	if err != nil {
		fmt.Println("Open Src File Failed err:", err)
		return
	}
	// 将文件读取出来,写入临时的切片中
	// 完整的元素的切片
	var data []byte
	// 每一次读取的元素,临时存储的切片
	var temp = make([]byte, 128)
	// 因为 temp 字节切片 有一定容量,一次读不完,所以可使用 for 循环读取
	for {
		n, err := src.Read(temp)
		// 判断文件是否已经读取完, 用 io.EOF
		if err == io.EOF {
			break
		}
		if err != nil {
			fmt.Println("Read src File Failed err:", err)
			break
		}
		// 利用 append 函数 将临时切片读取到的内容 追加到完整切片中
		data = append(data, temp[:n]...)
	}
	// 字节切片,打印的时候需要转换成 string 类型
	defer src.Close()
	// 以 写入|创建 的方式打开 目标文件
	dst, err := os.OpenFile(dstName, os.O_WRONLY|os.O_CREATE, 0644)
	if err != nil {
		fmt.Println("Open Dst File Failed err:", err)
		return
	}
	n, err = dst.Write(data)
	if err != nil {
		fmt.Println("Write Dst File Failed err:", err)
		return
	}
	defer dst.Close()
	fmt.Println("复制成功")
	return
}

func main() {
	n, err := copyFile("./xx.txt", "./temp.txt")
	if err != nil {
		fmt.Println("Copy File Failed err:", err)
		return
	}
	fmt.Printf("复制了 %d 行\n", n)
}
```


