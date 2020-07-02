# Golang sqlx模块


# Go Web 编程

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

