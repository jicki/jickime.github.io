# Golang 操作 Mysql


# Go Web 编程

## sql 标准库

* Go 语言中的 `database/sql` 包提供了保证SQL或类SQL数据库的范接口, 但是并不提供具体的数据库驱动。

* 使用`database/sql`包时 必须注入至少一个数据库驱动。


> 下载 Mysql 驱动

```shell

go get -u github.com/go-sql-driver/mysql


```


> 驱动说明


```go
// sql.Open 函数

func Open(driverName, dataSourceName string) (*DB, error)


``` 

* `driverName` - 指具体的数据库类型, 如: mysql

* `dataSourceName` - 指具体的数据信息, 如: 账号、密码、协议、IP、端口、数据库名 等。



> 例子

```go

package main

import (
	"database/sql"

        // 引入 mysql 驱动, 只需要用到 init() 方法
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	dsn := "user:password@tcp(127.0.0.1:3306)/dbname"
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
	// 关闭数据库连接
	defer db.Close()

	// Ping 函数 尝试校验 dsn
	err = DB.Ping()
	if err != nil {
		return err
	}
}

```


> 初始化连接的例子

* `sql.Open` 返回的 DB 对象可以安全的被多个`goroutine`并发使用, 并维护自己的空闲连接池, 因此很少需要关闭 DB 这个对象。


* `SetMaxOpenConns` 设置与数据库建立连接的最大连接数。默认为0 (无限制)


* `SetMaxIdleConns` 设置连接池中最大闲置连接数。



```go
package main

import (
	"database/sql"
	"fmt"

	// 引入 mysql 驱动, 只需要用到 init() 方法
	_ "github.com/go-sql-driver/mysql"
)

// 定义全局的 数据库对象 DB
var DB *sql.DB

func initDB() (err error) {
	dsn := "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True"

	// 这里 DB 赋值是给 上面定义的全局变量赋值, 不要使用 :=
	DB, err = sql.Open("mysql", dsn)
	if err != nil {
		return err
	}
	// 调用 Ping() 方法会尝试连接数据库,校检 dsn 是否正确
	err = DB.Ping()
	if err != nil {
		return err
	}
	// 最大连接数
	DB.SetMaxOpenConns(100)
	// 连接池中空闲连接数
	DB.SetMaxIdleConns(50)
	return nil
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
}

```






## sqlx 模块

> 官方地址 https://github.com/jmoiron/sqlx
>
> sqlx 是 go 标准库 `database/sql` 的一个扩展库, 在保持 `database/sql` 标准接口不变的情况下增加了一些扩展。

* sqlx 的扩展

  * 可将行记录映射给 `struct（内嵌struct也支持）`， map与slices

  * 支持在preprared statement 中使用命名参数

  * 将Get和Select的查询结果保存到`struct` 和 `slice` 中


## sqlx 模块使用

```shell
go get -u github.com/jmoiron/sqlx

```


### 插入数据库

```shell
/*
数据库 student 表

CREATE TABLE `student` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(64) NOT NULL,
  `nick` varchar(64) DEFAULT NULL,
  `country` varchar(128) DEFAULT NULL,
  `province` varchar(64) DEFAULT NULL,
  `city` varchar(64) DEFAULT NULL,
  `status` int(11) DEFAULT NULL,
  `create_time` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

*/
```

```go
import (
	"fmt"
	"log"

	"github.com/jmoiron/sqlx"

	_ "github.com/go-sql-driver/mysql"
)

// 创建一个学生结构体,并为 db 打上标签
type Student struct {
	Id         int    `db:"id"`
	Name       string `db:"name"`
	Nick       string `db:"nick"`
	Country    string `db:"country"`
	Province   string `db:"province"`
	City       string `db:"city"`
	ImgUrl     string `db:"img_url"`
	Status     int    `db:"status"`
	CreateTime string `db:"create_time"`
}

// 操作数据库
func main() {
	// 连接数据库信息
	dsn := "root:rldb@tcp(127.0.0.1:3306)/jicki?parseTime=true"
	// Connect 方法: 内部实现了Open()和Ping()
	DB, err := sqlx.Connect("mysql", dsn)
	if err != nil {
		log.Fatalln(err)
	}
	// 关闭数据库连接
	defer DB.Close()

	// 创建2个结构体实例 s1 , s2
	s1 := Student{
		Id:       1,
		Name:     "No1",
		Nick:     "Num1",
		Country:  "China",
		Province: "GD",
		City:     "SZ",
		Status:   1,
	}

	s2 := Student{
		Id:       2,
		Name:     "No2",
		Nick:     "Num2",
		Country:  "China",
		Province: "GD",
		City:     "SZ",
		Status:   1,
	}

	// 插入数据 一:
	sqlStr := `insert into student (id,name,nick,country,province,city,status,create_time)
	           values(?,?,?,?,?,?,?,?)`
        DB.MustExec(sqlStr, s2.Id, s2.Name, s2.Nick, s2.Country, s2.Province, s2.City, s2.Status, time.Now())

	// 插入数据 二: (:后面是数据库字段) , 最后传入整个 结构体

	_, err = DB.NamedExec(`INSERT INTO student VALUES (:id, :name, :nick, :country, :province, :city, :status, :create_time);`, s1)
	if err != nil {
		log.Fatalf("NamedExec Failed err:%v\n", err)
	}
	fmt.Println("MustExec OK !")
}

```

* 输出


```shell

MustExec OK !

```

### 查询数据

* 查询单条数据

```go
import (
        "fmt"
        "log"

        "github.com/jmoiron/sqlx"

        _ "github.com/go-sql-driver/mysql"
)

// 创建一个学生结构体,并为 db 打上标签
type Student struct {
        Id         int    `db:"id"`
        Name       string `db:"name"`
        Nick       string `db:"nick"`
        Country    string `db:"country"`
        Province   string `db:"province"`
        City       string `db:"city"`
        ImgUrl     string `db:"img_url"`
        Status     int    `db:"status"`
        CreateTime string `db:"create_time"`
}

// 操作数据库
func main() {
        // 连接数据库信息
        dsn := "root:rldb@tcp(127.0.0.1:3306)/jicki?parseTime=true"
        // Connect 方法: 内部实现了Open()和Ping()
        DB, err := sqlx.Connect("mysql", dsn)
        if err != nil {
                log.Fatalln(err)
        }
        // 关闭数据库连接
        defer DB.Close()

	// 查询单条数据 :
	// 创建一个空的结构体实例 s3:
	var s3 Student
	//将查询的结果存入 s3 空结构体指针中
	err = DB.Get(&s3, "SELECT id,name,nick,country,province,city,status,create_time FROM student WHERE name = ?", "No1")
	if err != nil {
		log.Fatalln(err)
	}
	// 打印查询的结构体
	fmt.Println(s3)
```

* 输出:

```shell

{1 No1 Num1 China GD SZ 1 }

```

* 查询多条数据

```go

import (
        "fmt"
        "log"

        "github.com/jmoiron/sqlx"

        _ "github.com/go-sql-driver/mysql"
)

// 创建一个学生结构体,并为 db 打上标签
type Student struct {
        Id         int    `db:"id"`
        Name       string `db:"name"`
        Nick       string `db:"nick"`
        Country    string `db:"country"`
        Province   string `db:"province"`
        City       string `db:"city"`
        ImgUrl     string `db:"img_url"`
        Status     int    `db:"status"`
        CreateTime string `db:"create_time"`
}

// 操作数据库
func main() {
        // 连接数据库信息
        dsn := "root:rldb@tcp(127.0.0.1:3306)/jicki?parseTime=true"
        // Connect 方法: 内部实现了Open()和Ping()
        DB, err := sqlx.Connect("mysql", dsn)
        if err != nil {
                log.Fatalln(err)
        }
        // 关闭数据库连接
        defer DB.Close()

	// 查询多条数据:
	// 定一个 结构体类型的 切片
	var studs []Student
	err = DB.Select(&studs, "SELECT id,name,nick,country,province,city,status,create_time FROM student")
	if err != nil {
		log.Fatalln(err)
	}
	// 循环打印结构
	for _, v := range studs {
		fmt.Printf("%#v \n", v)
	}

	// 利用 Query 查询
	// 创建一个空的Student的结构体
	s4 := Student{}
	rows, err := DB.Queryx("select id,name,nick,country,province,city,status,create_time FROM student")
	if err != nil {
		log.Fatal(err)
	}
	// Next 方法
	for rows.Next() {
		// 将查询的结构返回到 s4 结构体指针中
		err := rows.StructScan(&s4)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Printf("%#v\n", s4)
	}
```

* 输出:

```shell

main.Student{Id:1, Name:"No1", Nick:"Num1", Country:"China", Province:"GD", City:"SZ", Status:1, CreateTime:""} 
main.Student{Id:2, Name:"No2", Nick:"Num2", Country:"China", Province:"GD", City:"SZ", Status:1, CreateTime:"2019-11-27 03:12:18.489914"} 

main.Student{Id:1, Name:"No1", Nick:"Num1", Country:"China", Province:"GD", City:"SZ", Status:1, CreateTime:""}
main.Student{Id:2, Name:"No2", Nick:"Num2", Country:"China", Province:"GD", City:"SZ", Status:1, CreateTime:"2019-11-27 03:12:18.489914"}
```

