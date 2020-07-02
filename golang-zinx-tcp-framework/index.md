# Golang 轻量级TCP框架 Zinx


# Zinx

* Zinx 是一个基于Golang的轻量级并发服务器框架.

* [Github] https://github.com/aceld/zinx


## Zinx 架构图

 ![ZinX][1]


## Zinx 使用

1. 创建Server句柄

2. 配置自定义`Router`路由以及定义业务功能

3. 启动服务


###  Server 端

* 例子:

```go
package main

import (
	"fmt"
	"zinx/ziface"
	"zinx/znet"
)

// 自定义路由

// 1. 创建一个结构体
type PingRouter struct {
	// 要使用路由 首先继承 BaseRouter 这个结构体
	znet.BaseRouter
}

/* 
  2. 重写 Router 的方法 (1. PreHandle 2. Handle  3. PostHandle)
     初始PreHandle, Handle, PostHandle 方法没有实现任何业务。
     可重写任何一个方法, 完成指定的业务。
*/

// 3. 重写 Handle 方法
func (p *PingRouter) Handle(request ziface.IRequest) {
	// 读取客户端的数据
	fmt.Println("recv from client: msgId=", request.GetMsgID(), ", data=", string(request.GetData()))

        // 回写数据到服务端
        err := request.GetConnection().SendBuffMsg(0, []byte("ping...ping...ping"))
	if err != nil {
		fmt.Println(err)
	}
}

// 启动服务
func main() {

	// 1. 创建一个Server 句柄
	s := znet.NewServer()

	// 2. 配置自定义的 Router 路由
	s.AddRouter(0, &PingRouter{})

	// 3. 开启服务
	s.Serve()
}

```



### Client

* Zinx 的消息处理采用 `[MsgLength]` 、 `[MsgID]` 、 `[Data]` 的封包格式。

```go
package main

import (
	"fmt"
	"io"
	"net"
	"time"
	"zinx/znet"
)

/*
	模拟客户端
 */
func main() {

	fmt.Println("Client Start ...")

	// 等待 3 秒之后发起测试请求, 给服务端开启服务的机会
	time.Sleep(3 * time.Second)

	// 初始化连接 服务端的 conn 句柄 
	conn,err := net.Dial("tcp", "127.0.0.1:8888")
	if err != nil {
		fmt.Println("Client Start err, exit!")
		return
	}

	for i := 0; i < 3; i++ {
		// 发封包message消息
		dp := znet.NewDataPack()
                 
		msg, _ := dp.Pack(znet.NewMsgPackage(0, []byte("Zinx Client Test Message")))
		_, err := conn.Write(msg)
		if err !=nil {
			fmt.Println("write error err ", err)
			return
		}

		// 读取流中的 head 部分
		headData := make([]byte, dp.GetHeadLen())
                // ReadFull 会把msg填充满为止 
		_, err = io.ReadFull(conn, headData)
		if err != nil {
			fmt.Println("read head error")
			break
		}
		//将headData字节流 拆包到msg中
		msgHead, err := dp.Unpack(headData)
		if err != nil {
			fmt.Println("server unpack err:", err)
			return
		}

		if msgHead.GetDataLen() > 0 {
			//msg 是有data数据的，需要再次读取data数据
			msg := msgHead.(*znet.Message)
			msg.Data = make([]byte, msg.GetDataLen())

			//根据dataLen从io中读取字节流
			_, err := io.ReadFull(conn, msg.Data)
			if err != nil {
				fmt.Println("server unpack data err:", err)
				return
			}

			fmt.Println("==> Recv Msg: ID=", msg.Id, ", len=", msg.DataLen, ", data=", string(msg.Data))
		}

		time.Sleep(time.Second)
	}
}
```

### Zinx 配置文件

```shell
{
  "Name":"zinx demoApp",
  "Host":"127.0.0.1",
  "TcpPort":8888,
  "MaxConn":3,
  "WorkerPoolSize":10,
  "LogDir": "./mylog",
  "LogFile":"zinx.log"
}

```

* `Name`: 服务器应用名称

* `Host`: 服务器IP

* `TcpPort`: 服务器监听端口

* `MaxConn`: 允许的客户端链接最大数量

* `WorkerPoolSize`: 工作任务池最大工作`Goroutine`数量

* `LogDir`: 日志文件夹

* `LogFile`: 日志文件名称(如果不提供, 则日志信息打印到Stderr)



  [1]: https://jicki.cn/img/posts/zinx/zinx.png

