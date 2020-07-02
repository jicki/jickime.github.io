# 全局唯一ID生成器 Sonyflake


# Go Web 编程


## Sonyflake

> 官方地址 github.com/sony/sonyflake

* `Sonyflake`是Sony公司基于 Twitter `snowflake` 实现的的一个Golang的全局性唯一ID生成器的开源项目, 基本思路和`snowflake` 差不多, 但也稍有不同。

* 如下为 snowflake 的ID 组成

|||||
|-|-|-|-|
|1 Bit Unused|41 Bit Timestamp|10 Bit Node ID|12 Bit Sequence ID|


* 如下为 Sonyflake 的ID 组成

|||||
|-|-|-|-|
|1 Bit Unused|39 Bit Timestamp|8 Bit Sequence ID|16 Bit Machine ID|


* 名词解析

  * Unused: 未使用, 二进制中最高位为1的都是负数, 我们生成的id一般都是整数, 所以这个最高位固定是0。

  * Timestamp: 时间戳. 

  * Sequence ID: 由程序在运行期生成。

  * Node ID: 机器编号的位长。

  * Machine ID:  其实就是 Node ID。


### 安装

```shell
go get -u github.com/sony/sonyflake

```

### import

```shell

import "github.com/sony/sonyflake"

```

### 创建 Sonyflake 实例

```go
func NewSonyflake(st Settings) *Sonyflake
```


### Setting 结构体

```go

type Settings struct {
	StartTime      time.Time
	MachineID      func() (uint16, error)
	CheckMachineID func(uint16) bool
}

```

* 说明:

  * StartTime: 将 `Sonyflake` 的时间定义为开始的时间, 如果`StartTime`为`0`, 则将`Sonyflake`的开始时间设置为 `2014-09-01 00:00:00 +0000 UTC` 。 如果`StartTime`早于当前时间, 则不会创建`Sonyflake`。

  * MachineID: 返回`Sonyflake`实例的唯一ID。 如果`MachineID` 返回错误，则不会创建`Sonyflake`。如果`MachineID`为nil, 则使用默认的`MachineID`。默认的`MachineID`返回专用IP地址的低16位。

  * CheckMachineID: 验证计算机ID的唯一性。如果`CheckMachineID`返回`false`，则不会创建`Sonyflake`。如果`CheckMachineID`为nil，则不进行验证。 



### 生成唯一ID

* 调用 NextID() 方法

```go
func (sf *Sonyflake) NextID() (uint64, error)

```

* 从 `StartTime` 设置的时间开始, 执行 NextID 生成ID 可以生成约 174年。但是在`Sonyflake`时间超过限制后, NextID将返回错误。


### 模拟生成ID例子

```go

package main

import (
	"fmt"
	"log"

	"github.com/sony/sonyflake"
)

var (
	sonyFlake *sonyflake.Sonyflake
	// 定义一个全局的 machineID 模拟获取
	// 现实环境中应从 zk 或 etcd 中获取
	machineID uint16
)

// 获取 机器编码ID的 回调函数
func getMachineID() (uint16, error) {
	// machineID 返回nil, 则返回专用IP地址的低16位
	return machineID, nil
}

// 初始化 sonyFlake 配置
func Init(mID uint16) (err error) {
	machineID = mID
	st := sonyflake.Settings{}
	sonyFlake = sonyflake.NewSonyflake(st)
	return
}

// 获取全局 ID 的函数
func GetID() (id uint64, err error) {
	if sonyFlake == nil {
		err = fmt.Errorf("需要先初始化以后再执行 GetID 函数 err: %#v \n", err)
		return
	}
	return sonyFlake.NextID()
}
func main() {
	mID, err := getMachineID()
	if err != nil {
		log.Fatalf("getMachineID Failed Err: %#v\n", err)
	}
	if err = Init(mID); err != nil {
		log.Fatalf("Init Err: %#v\n", err)
	}
	id, err := GetID()
	if err != nil {
		log.Fatalf("GetID Failed Err: %#v\n", err)
	}
	fmt.Println("sonyFlake 生成 ID: ", id)
}

```

* 输出:


```shell

sonyFlake 生成 ID:  280652487722051811

```

