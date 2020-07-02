# 数据传输协议 Protocol Buffer


# Protocol Buffer


## 数据传输协议


* 一般常用的数据传输协议为 `json`, `xml`, `Protobuf` 。

  * `json`

    * 优势: 可读性强, 直观, 支持跨平台多语言。
    * 劣势: 编解码很耗时。

  * `xml`

    * 优势: 基于标签形式。
    * 劣势: 局限性 一般在 前端/页面 中使用, 可读性不强, 冗余性不太好。

  * `Protobuf`

    * 优势: 编解码速度快, 序列化后体积比`json`/`xml` 小, 适用于网络传输, 支持跨平台多语言。 
    * 劣势: 可读性不强, 二进制格式、不是明文传输, 注释描述差。



## 配置 Protobuf


### 安装 protoc

* 直接下载官方编译好的二进制文件,解压后复制到 `$GOPATH/bin` 目录下

```shell

https://github.com/google/protobuf/releases

# Win   下载以 protoc-3.11.2-win64.zip 的二进制文件
# Linux 下载以 protoc-3.11.2-linux-x86_64.zip 的二进制文件
# MacOS 下载以 protoc-3.11.2-osx-x86_64.zip 的二进制文件
```

```shell
# protoc --version
libprotoc 3.11.2

```


### 安装 protoc-gen-go

```shell
go get -u -v  github.com/golang/protobuf/protoc-gen-go

```

* go get `protoc-gen-go` 以后会自动build 生成二进制文件放到 `$GOPATH/bin` 目录下



### 安装 protobuf 库

```shell
go get -u -v github.com/golang/protobuf/proto

```

## protobuf 数据类型


|proto|Go|说明|
|-|-|-|
|double|float64|64位浮点数|
|float|float32|32位浮点数|
|int32|int32|对于可变长编码,负数效率低, 请使用 sin64 代替.|
|uint32|uint32|无符号32位整型|
|uint64|uint64|无符号64位整型|
|sint32|int32|存在负值时,效率比int32效率高很多|
|sint64|int64|存在负值时,效率比int64效率高很多|
|fixed32|uint32|总是4个字节, 如果总和比228大,这个类型比uint32高效|
|fixed64|uint64|总是8个字节, 如果总和比256打,这个类型比uint64高效|
|sfixed32|int32|总是4个字节|
|sfixed64|int64|总是8个字节|
|bool|bool|布尔值|
|string|string|字符串类型，并且必须是UTF-8编码或者7-bit ASCII编码|
|bytes|[]byte|包含任意顺序的字节数据|




## protobuf 语法


* protobuf 文件一般约定为 proto 为后缀,如: `Person.proto`


```shell
// 必须指定版本信息,否则报错
syntax = "proto3";

// 生成go文件的包名
package pb;

// 定义一个消息类型,使用 message 关键字
message Person {
    string name = 1;
    int32 age = 2;
    // repeated 表示字段允许重复
    repeated string emails = 3;
    repeated PhoneNumber phones = 4;
}




// 定义一枚举类型,使用 enum 关键字
enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
}

// 消息类型嵌套
message PhoneNumber {
    string number = 1;
    // PhoneType 是 enum 定义的枚举类型, 只允许取枚举中定义的三个值。
    PhoneType type = 2;
}

```


## 编译 protobuf 文件

* `protoc --proto_path=IMPORT_PATH --go_out=DST_PATH /path/file.proto`

```shell
# 这里也可以使用 protoc --go_out=. *.proto  简化操作.
# 切换到 存放 proto 文件的目录下

protoc --go_out=. *.proto

# 生成 .go 文件, 此文件不能修改。

Person.pb.go 

```


## Golang 使用 protobuf


```go

package main

import (
	"fmt"
        // proto 文件生成 go 文件存放的路径
	"jicki/protobuf_demo/pb"

	"github.com/gogo/protobuf/proto"
)

//
func main() {
	// 定义一个 Person 结构体
	person := &pb.Person{
		Name:   "小炒肉",
		Age:    20,
		Emails: []string{"jicki@qq.com", "jicki@vip.qq.com"},
		Phones: []*pb.PhoneNumber{
			&pb.PhoneNumber{
				Number: "1333333333",
				Type:   pb.PhoneType_MOBILE,
			},
			&pb.PhoneNumber{
				Number: "85858585",
				Type:   pb.PhoneType_HOME,
			},
		},
	}
	// 序列化:
	// 将 person 结构体, 进行 proto 序列化, 得到一个二进制文件.
	// 如下: data 为需要传输的 数据
	data, err := proto.Marshal(person)
	if err != nil {
		fmt.Println("proto Marshal err: ", err)
		return
	}

	// 反序列化:
	// 定义一个空结构体对象
	newPerson := &pb.Person{}
	err = proto.Unmarshal(data, newPerson)
	if err != nil {
		fmt.Println("proto Unmarshal err: ", err)
		return
	}

	fmt.Println("源数据: ", person)

	fmt.Println("序列化后的数据: ", data)

	fmt.Println("反序列化后的数据: ", newPerson)
}

```

* 输出:

```shell
源数据:  name:"\345\260\217\347\202\222\350\202\211" age:20 emails:"jicki@qq.com" emails:"jicki@vip.qq.com" phones:<number:"1333333333" > phones:<number:"85858585" type:HOME > 

序列化后的数据:  [10 9 229 176 143 231 130 146 232 130 137 16 20 26 12 106 105 99 107 105 64 113 113 46 99 111 109 26 16 106 105 99 107 105 64 118 105 112 46 113 113 46 99 111 109 34 12 9 51 51 51 51 51 51 51 51 51 34 12 10 8 56 53 56 53 56 53 56 53 16 1]

反序列化后的数据:  name:"\345\260\217\347\202\222\350\202\211" age:20 emails:"jicki@qq.com" emails:"jicki@vip.qq.com" phones:<number:"1333333333" > phones:<number:"85858585" type:HOME > 

```


