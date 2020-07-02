# Go学习笔记


# Go语言基础

## Go基础结构

```go
// 程序的入口
package main  

// 导入的包
import "fmt"

// main 函数的内容,每个程序必须包含 main 函数作为入口
func main() {
    // 单行注释写法
    /*
    多行注释,块注释
    */
    // 执行fmt包下的Println函数
	fmt.Println("hello jicki!")
}
```

* go 程序执行
  * 第一种 go run main.go
  * 第二种(发布程序) go build , 然后再执行 ./main


