# Go学习笔记


# Go语言基础

## 引用类型与值类型

### 值类型

* 值类型有: 整型int、浮点型float、布尔值bool、字符串string、数组Array、结构体struct

* 值类型的特点是: 变量直接存储值, 内存通常在栈中分配.


```go

package main

import "fmt"

func main(){
    // 定义一个数组 a , 数组是值类型
    a := [5]int{2, 3, 4, 5, 6}
    // a 复制给 b 是 copy
    b := a
    fmt.Println(a,b)
    // 当b 发生改变 a 的值不会发生改变
    b[2] = 77
    fmt.Println(a,b)
}

```

* 输出

```go
[2,3,4,5,6] [2,3,4,5,6]
[2,3,4,5,6] [2,3,77,5,6]

```

### 引用类型

* 引用类型有: 指针pointer、切片slice、管道channel、接口interface、map、函数func

* 引用类型的特点是: 变量存储的是一个地址, 这个地址对应的空间里才是真正存储的值, 内存通常在堆中分配.


```go
package main

import "fmt"

func main(){
    // 定义一个 切片 a , 切片是引用类型
    a := []int{2, 3, 4, 5, 6}
    // a 复制给 b 是 指针的引用, a 与 b 指向同一个底层数组
    b := a
    fmt.Println(a,b)
    // 当 b 发生改变时, a 的值也会跟着改变
    b[2] = 77
    fmt.Println(a,b)
}

```

* 输出:

```go
[2,3,4,5,6] [2,3,4,5,6]
[2,3,77,5,6] [2,3,77,5,6]

```

