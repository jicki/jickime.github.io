# Go学习笔记


# Go语言基础

## 数组Array

* 数组是同一种数据类型元素的集合。 在Go语言中，数组从声明时就确定，使用时可以修改数组成员，但是数组大小不可变化。

* 数组的定义: `var 数组变量名 [元素数量]类型` (如: var a [3]int )

* 数组的长度必须是常量, 并且长度是数组类型的一部分。一旦定义, 长度不能变。

* 数组可以通过索引进行访问,但是不能超过数组元素最大索引,否则会引发 `panic`

### 数组定义

```go
func main() {
	// 数组的定义
	var a1 [3]int
	var a2 [4]int
	// 数组的赋值, 未赋值的数组是对应类型元素的0值
	a1[0] = 1
	fmt.Println(a1)
	fmt.Println(a2)

}
```

### 数组初始化

```go
func main() {
    // 数组初始化
	// 定义数组时初始化元素
	var city = [4]string{"北京", "上海", "广州", "深圳"}
	fmt.Println(city)

	// 定义数组时使用[...]编译器动态配置数组长度
	var number = [...]int{1, 2, 3, 4, 5, 6, 7, 8}
	fmt.Println(number)

	// 使用索引值方式初始化
	var stu = [...]string{0: "stu1", 1: "stu1", 3: "stu3", 5: "stu5"}
	fmt.Printf("%#v\n", stu)
}

```

### 数组遍历

* 数组遍历可使用 for 循环与 for range 循环进行遍历

```go
func main() {
    	// 数组遍历
	a3 := [...]string{"北", "上", "广", "深"}

	// 利用 for 循环,使用数组索引进行遍历数组元素
	for i := 0; i < len(a3); i++ {
		fmt.Println(a3[i])
	}

	// 利用 for range 循环进行遍历数组的索引与元素
	for i, k := range a3 {
		fmt.Println(i, k)
	}
}
```

### 多维数组

* 多维数组既,数组中嵌套数组
* 多维数组只有最外层数组可以使用[...]动态配置数组长度
* 数组是值类型, 赋值和传参会复制整个数组. 因此改变副本的值, 不会改变本身的值。

```go
func main() {
    // 多维数组
	a4 := [3][2]string{
		{"北京", "南京"},
		{"深圳", "广州"},
		{"上海", "杭州"},
	}
	fmt.Printf("%#v \n", a4)

	// 多维数组的遍历
	for _, v1 := range a4 {
		for _, v2 := range v1 {
			fmt.Println(v2)
		}
    }
  
    // 数组是值类型
	a5 := [...]int{1, 2, 3, 4, 5}
	y := a5
	y[0] = 100
	// 不会修改原数组的值
	fmt.Println(a5)
}

```

### 数组练习题

```go

func main() {
	// 1. 求数组[1, 3, 5, 7, 8]所有元素的和
	a1 := [...]int{1, 3, 5, 7, 8}
	sum := 0
	for _, k := range a1 {
		sum = sum + k
		fmt.Println(sum)
	}
	// 2. 找出数组中和为指定值的两个元素的下标，比如从数组[1, 3, 5, 7, 8]中找出和为8的两个元素的下标
	// 第一种 for range
	a2 := [...]int{1, 3, 4, 5, 7, 8, 9}
	for index, value := range a2 {
		for i := 0; i < len(a2)/2; i++ {
			if value+a2[i] == 8 {
				// 元素值
				//fmt.Println(value, a2[i])
				// 下标
				fmt.Println(index, i)
			}
		}
	}
	// 第二种 for
	// 第一次遍历数,取全部元素
	for i := 0; i < len(a2); i++ {
		// 第二次遍历数组,只取一半元素
		for j := 0; j < len(a2)/2; j++ {
			if a2[i]+a2[j] == 8 {
				fmt.Println(i, j)
			}
		}
	}
}
```

## 切片Slice

* 切片 (Slice) 是一个拥有相同类型元素的可变长度的序列。它是基于数组类型做的一层封装。它非常灵活，支持自动扩容。

* 切片 (slice) 是一个`引用类型`, 它的内部结构包含地址、长度和容量。切片一般用于快速地操作一块数据集合。

* 切片 (slice) 定义了以后, 必须初始化才能使用。

* 切片 (slice) 不能直接进行比较, 切片只能和 `nil` 进行比较. `nil` 的切片元素个数与容量都为 0。

### 切片定义

* `var 变量名 []元素类型`.

```go
func main() {
	// 切片的定义
	var s1 []string
	var s2 []int
	// 定义时就初始化
	var s3 = []bool{false, true}
	// 没有初始化的 切片元素为 nil
	fmt.Printf("s1 = %#v s2 = %#v  s3 = %#v \n", s1, s2, s3)
	// 基于数组定义切片
	a1 := [5]int{1, 2, 3, 4, 5}
	// 左包含右不包含, 既 元素下表为 a1[1],a1[2],a1[3] 的元素
	s4 := a1[1:4]
	fmt.Printf("a1 = %#v  s4 = %#v \n", a1, s4)
	// 基于 make 函数构造切片 (如下 5 = 元素个数, 10 = 容量)
	s5 := make([]int, 5, 10)
	fmt.Printf("s5 len = %d  s5 cap = %d \n", len(s5), cap(s5))
}
```

### 切片本质

* 切片的本质就是对底层数组的封装, 它包含了三个信息: 底层数组的指针、切片的长度（len）和切片的容量（cap）。

```go
func main() {
	// 定义了一个 元素个数8,容量为8 的切片(没有定义容量时,容量等于元素个数)
	s6 := []int{1, 2, 3, 4, 5, 6, 7, 8}
	fmt.Printf("s6 len = %d s6 cap = %d \n", len(s6), cap(s6))
	// 在切片 s6 的基础上 定义的一个切片, 元素个数为 2 (s6[2],s6[3])
	// 容量 等于 s6 容量 减去 前面2个元素 s7 容量为 6
	s7 := s6[2:4]
	fmt.Printf("s7 len = %d s7 cap = %d \n", len(s7), cap(s7))
}
```

### 切片比较

```go
func main() {
	// 切片不能直接比较,只能与 nil 值进行比较
	var s8 []int
	// nil 值的切片 元素与容量都为 0
	fmt.Printf("s8 len = %d s8 cap = %d \n", len(s8), cap(s8))
	// 与 nil 比较, 判断为 true 进行初始化
	if s8 == nil {
		fmt.Printf("%#v \n", s8)
		fmt.Println("s8 == nil")
		s8 = make([]int, 5, 10)
		fmt.Printf("%#v \n", s8)
	}
}
```

### 切片的赋值

* 切片是 引用类型

```go
func main() {
	// 切片的赋值与拷贝

	// 定义个初始化的切片,元素个数为 5
	s9 := make([]int, 5)
	// 这里 s9, s10 指向一个相同的 内存地址
	s10 := s9
	fmt.Println(s9)
	fmt.Println(s10)
	// 对 s10[0] 个元素进行赋值
	s10[0] = 10
	// 如下两个切片的元素都被修改了
	fmt.Println(s9)
	fmt.Println(s10)
}

```

### 切片的遍历

* 切片的遍历方式和数组是一致的, 支持索引`for`遍历和`for range`遍历。

```go
func main() {
	// 切片的遍历
	s11 := []string{"北京", "上海", "广州", "深圳"}
	// for 循环
	for i := 0; i < len(s11); i++ {
		fmt.Println(s11[i])
	}
	// for range 循环
	for i, k := range s11 {
		fmt.Println(i, k)
	}
}

```

### append()

* Go语言的内建函数`append()`可以为切片动态添加元素。 每个切片会指向一个底层数组, 这个数组能容纳一定数量的元素。当底层数组不能容纳新增的元素时, 切片就会自动按照一定的策略进行"扩容", 此时该切片指向的底层数组就会更换。"扩容"操作往往发生在`append()`函数调用时.

```go
func main() {
	// append() 方法
	// 使用 append 方法 不需要额外初始化
	var s12 []int
	s12 = append(s12, 100)
	fmt.Println(s12)
	// 切片自动扩容容量,相同容量情况下内存地址不变,扩容以后内存地址变化.
	for i := 0; i < 5; i++ {
		fmt.Printf("s12 len = %d cap = %d ptr = %p\n", len(s12), cap(s12), s12)
		s12 = append(s12, i)
	}
	// append 在 切片中 追加 切片
	s13 := []int{6, 7, 8}
	// 追加的切片后面必需要加 ... (s13... 表示将切片的元素一个一个取出来)
	s12 = append(s12, s13...)
	fmt.Println(s12)

	// 利用 append 实现删除切片某个元素
	// 删除 s13 切片中 s13[1] 个元素 "7"
	s13 = append(s13[:1], s13[2:]...)
	fmt.Println(s13)
}

```

### 切片扩容策略

* 切片容量长度小于1024时, 每次扩容的新容量就是旧容量的2倍。
* 如果新扩容的容量大于旧容量的2倍, 那么旧容量就等于新扩容的容量。
* 如果切片容量长度大于1024时,扩容的新容量是旧容量的1/4, 直到最终容量大于等于新申请的容量。
* 另外不同类型的切片, 扩容的策略也不相同。

### copy()

* 由于切片是`引用类型`, 所以切片(a = b) 实际指向的是同一个内存地址,修改切片b的元素时,a切片也会修改。

* Go语言内建的`copy()`函数可以迅速地将一个切片的数据复制到另外一个切片空间中。

* `copy()` 用法 `copy(目标切片, 原切片)`。

```go
func main() {
	// copy() 切片的复制
	s14 := []int{1, 2, 3, 4, 5}
	s15 := make([]int, 5)
	s16 := make([]int, 5)
	// 直接赋值
	s15 = s14
	// 使用 copy 函数
	copy(s16, s14)
	// 修改 s15 的值
	s15[0] = 20
	// 修改 s16 的值
	s16[0] = 10
	// 打印 s14 的值 与 内存地址
	fmt.Printf("s14 = %d  s14 prt = %p \n", s14, s14)
	// 打印 s15 的值 与 内存地址
	fmt.Printf("s15 = %d  s15 prt = %p \n", s15, s15)
	// 打印 s16 的值 与 内存地址
	fmt.Printf("s16 = %d  s16 prt = %p \n", s16, s16)
}

```

### 切片练习题

```go
func main() {
	// 练习题 一
	// 定义一个 string类型 的切片 a 元素长度5,容量10
	var a = make([]string, 5, 10)
	// a = [     ] string类型 元素为 5个空值
	for i := 0; i < 10; i++ {
		// 循环追加元素到 a 切片中,(fmt.Sprintf 将int类型转换成 string类型)
		a = append(a, fmt.Sprintf("%v", i))
	}
	// [      0 1 2 3 4 5 6 7 8 9] 前面5个空值
	fmt.Println(a)

	// 练习题 二
	// 使用内置的sort包对数组 var a1 = [...]int{3, 7, 8, 9, 1} 进行排序。
	var a1 = [...]int{3, 7, 8, 9, 1}
	// 将 数组 转换成 切片,再使用 sort.Ints() 进行排序
	sort.Ints(a1[:])
	fmt.Println(a1)
}

```

## Map

* Go语言中提供的映射关系容器为`map` , 其内部使用散列表`(hash)`实现。

* `map`是一种无序的基于`key-value`的数据结构, Go语言中的`map`是引用类型, 必须初始化才能使用。

* `map`类型的变量默认初始值为`nil`，需要使用`make()`函数来分配内存。

### map定义

* `map[KeyType]ValueType` (KeyType:表示键的类型, ValueType:表示键对应的值的类型) 如: map[string]int

```go
func main() {
	// map 定义
	// 定义一个map,未初始化(nil)
	var m1 map[string]int
	// 定义个map并初始化 元素类型的零值 ({})
	var m2 = make(map[string]int, 10)
	// 给 map 添加键值对(赋值), 必须初始化才能添加键值对
	m2["name"] = 1
	m2["age"] = 20
	m2["score"] = 80

	// 定义一个map并初始化和赋值 ({"age":10, "score":90})
	var m3 = map[string]int{
		"age":   10,
		"score": 90,
	}

	fmt.Printf("%#v \n", m1)
	fmt.Printf("%#v \n", m2)
	fmt.Printf("%#v \n", m3)
}

```

* 输出:

```go
map[string]int(nil)
map[string]int{"age":20, "name":1, "score":80}
map[string]int{"age":10, "score":90}
```

### map 初始化

* `make(map[KeyType]ValueType, [cap])` 如: make(map[string]int,10) , 其中`cap`表示`map`的容量, 该参数虽然不是必须的, 但是我们应该在初始化`map`的时候就为其指定一个合适的容量。

```go
func main(){
	// 定义一个 map, 未初始化(nil)
	var m1 map[string]int
	// 初始化 map, 使用make函数,容量为 10
	m1 = make(map[string]int, 10)
	fmt.Printf("%#v \n", m1)
}
```

* 输出:
  
```go
map[string]int{}

```

### 判断map键值对是否存在

* 使用`value`, `ok` 的方式进行判断。

```go
	// 判断 map 某个键值 是否存在 m4 中,使用 value,ok 的方式判断
	value, ok := m4["张三"]
	if ok {
		fmt.Printf("张三 value = %d \n", value)
	} else {
		fmt.Println("查无此人")
	}

	value, ok = m4["李六"]
	if ok {
		fmt.Printf("李六 = %d \n", value)
	} else {
		fmt.Println("查无此人")
	}
```

* 输出:

```go
张三 value = 20
查无此人
```

### map 的遍历

* 使用 `for range` 循环遍历 map

```go
func main() {
	// map 的遍历
	m5 := map[string]int{
		"哈哈": 20,
		"呵呵": 30,
		"嘻嘻": 40,
	}
	// for range 循环
	for key, value := range m5 {
		// 由于 map 是无序的, 所以输出的结果顺序与添加顺序会不一致。
		fmt.Println(key, value)
	}
	// 只获取key或者value
	for key := range m5 {
		fmt.Println(key)
	}
	for _, value := range m5 {
		fmt.Println(value)
	}
}
```

* 输出:

```go
哈哈 20
呵呵 30
嘻嘻 40
```

### delete() 函数

* `map` 可使用`delete()`内建函数 删除一组键值对, `delete()`函数。
* 格式: `delete(map, map的key)`

```go
func main() {
	// 使用 delete() 函数删除 map 中的键值对
	m6 := map[string]int{
		"a": 1,
		"b": 2,
		"c": 3,
		"d": 4,
	}
	// 删除键值对 "a"
	delete(m6, "a")
	fmt.Println(m6)
}
```

* 输出:

```go
map[b:2 c:3 d:4]
```

### 按照指定顺序遍历 map

* 将`map`中的 key 值存到切片中,利用 sort 对切片进行排序.然后利用 map[排序后的key] 对`map`进行排序.

```go
func main() {
	// 按照固定顺序遍历 map
	m7 := make(map[string]int, 50)
	// 使用 for 循环 增加 30个键值对
	for i := 0; i < 30; i++ {
		// 使用 Sprintf 生成 29个 key
		key := fmt.Sprintf("stu%02d", i)
		// 使用 rand 生成 29个 随机数
		value := rand.Intn(100)
		m7[key] = value
	}
	// 将 map 的 key 值保存到 slice 切片中
	// 定义一个 []string 类型的切片, 长度为0,容量为50
	keys := make([]string, 0, 50)
	// 使用 append()函数 将 map 中的 key 元素 存到 切片中
	for key := range m7 {
		keys = append(keys, key)
	}
	// 利用 sort.Strings 对 string 类型 进行排序
	sort.Strings(keys)
	// 利用排序后的 keys 对 map 排序
	for _, key := range keys {
		// _ = keys切片中的 索引
		// key = keys切片中的 键值
		fmt.Println(key, m7[key])
	}
```

### 切片与map结合

* 元素类型 为 `map` 的切片, `s1 = make([]map[string]int, 0, 5)`。

```go
func main() {
	// 元素类型为 map 的 切片
	// 对 切片 初始化 元素5, 容量5
	s1 := make([]map[string]int, 5, 5)
	// 对切片 初始化后 还需要对 map 的每个元素(key) 进行初始化.
	s1[0] = make(map[string]int, 5)
	// 初始化后进行赋值
	s1[0]["age"] = 6
	// 可以利用 == nil 的方式进行判断初始化
	for i := 0; i < len(s1); i++ {
		if s1[i] == nil {
			s1[i] = make(map[string]int, 5)
			s1[i]["age"] = 5
		}
	}
	fmt.Println(s1)
}
```

* 输出:

```go
[map[age:6] map[age:5] map[age:5] map[age:5] map[age:5]]
```

* 元素值为切片的 `map` 如: `m8 := make(map[string][]int, 8)`

```go
func main(){
	// 元素值为切片的 map
	// 定义一个 map 并初始化了 map, 容量为 8
	m8 := make(map[string][]int, 8)
	// 对 map 初始化以后, 还需要对 元素值 []int 进行初始化
	// 利用 v, ok 语法判断 map 的key是否存在
	v, ok := m8["MapName"]
	if ok {
		fmt.Println(v)
	} else {
		// 判断不存在 就创建键值对
		// 由于这里 map 的 value 是 []int 所以还需要对 切片进行初始化
		m8["MapName"] = make([]int, 5, 5)
		// 赋值
		m8["MapName"][0] = 100
		m8["MapName"][1] = 200
	}
	fmt.Println(m8)
}
```

* 输出:

```go
map[MapName:[100 200 0 0 0]]
```

### Map练习题

* 统计一个字符串中每个单词出现的次数。比如: "how do you do" 中how=1 do=2 you=1。

```go
	// 练习题
	// 统计 "how do you do" 一个字符串中每个单词出现的次数。
	// 定义一个 map key是 string 用于存放 字符串, value 是 int 类型用于记录出现的次数。
	str := "how do you do"
	m9 := make(map[string]int, 10)
	// 利用 strings.Split 对整个字符串进行分割
	words := strings.Split(str, " ")
	// 遍历 分割后的切片,取到每个元素
	for _, word := range words {
		// 利用 ok 来判断 map 中是否存在这个 key 值
		v, ok := m9[word]
		if ok {
			// 如果 map 中存在这个键值, value 就 + 1
			m9[word] = v + 1
		} else {
			// 如果 map 中不存在这个 键值, 创建一个键值, value = 1
			m9[word] = 1
		}
	}
	fmt.Println(m9)
```

* 输出:

```go
map[do:2 how:1 you:1]
```


