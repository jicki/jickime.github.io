# Golang 操作 Mysql


# Go Web 编程

## sql 标准库

* Go 语言中的 `database/sql` 包提供了保证SQL或类SQL数据库的范接口, 但是并不提供具体的数据库驱动。

* 使用`database/sql`包时 必须注入至少一个数据库驱动。


### 下载 Mysql 驱动

```shell

go get -u github.com/go-sql-driver/mysql


```


### 驱动说明


```go
// sql.Open 函数

func Open(driverName, dataSourceName string) (*DB, error)


``` 

* `driverName` - 指具体的数据库类型, 如: mysql

* `dataSourceName` - 指具体的数据信息, 如: 账号、密码、协议、IP、端口、数据库名 等。



### 连接数据库例子

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


### 初始化连接的例子

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


### 增删改查操作


> 查询操作


* 单行查询 - 单行查询使用`DB.QueryRow()` 方法执行一次查询, 并返回最多一行结果(Row)

  * `QueryRow`方法 总是返回非`nil`值, 直到返回值的`Scan`方法被调用时才会返回被延迟的错误. 如: 未查询到结果

  * `func (db *DB) QueryRow(query string, args ...interface{}) *Row` 方法



```go

type user struct {
	id   int
	name string
	age  int
}

func queryRowToID(id int) {
	var u user
	sqlStr := "select id, name, age from user where id=?"
        // .Scan 是链式操作, Scan是将查询出来的结果赋值到结构体对应的字段中.
        // 使用 QueryRow 以后必须要调用 Scan 方法,否则数据库连接不会被释放.
	err := DB.QueryRow(sqlStr, id).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("QueryRow Scan Failed, Error %v\n", err)
		return
	}
	fmt.Printf("ID: %d Name: %s  Age: %d \n", u.id, u.name, u.age)
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	queryRowToID(1)
}


```

```shell
# 输出结果
ID: 1 Name: jicki  Age: 20 
```


* 多行查询 - 多行查询使用 `DB.Query()` 方法执行一次查询, 返回多行结果(Row)。

  * `func (db *DB) Query(query string, args ...interface{}) (*Rows, error)` 方法



```go

type user struct {
	id   int
	name string
	age  int
}

func queryMultiRowToID(id int) {
	var u user
	sqlStr := "select id, name, age from user where id > ?"
	rows, err := DB.Query(sqlStr, id)
	if err != nil {
		fmt.Printf("QueryMulti Failed, Error %v\n", err)
		return
	}
	// 关闭数据库连接
	defer rows.Close()

	// 遍历读取的多条数据
	for rows.Next() {
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("QueryMulti Scan Failed, Error %v\n", err)
			return
		}
		fmt.Printf("ID: %d Name: %s  Age: %d \n", u.id, u.name, u.age)
	}
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	queryMultiRowToID(0)
}

```


```shell
# 输出结果
ID: 1 Name: jicki  Age: 20 
ID: 2 Name: Jack  Age: 30 
ID: 3 Name: Vicki  Age: 30 
```



> 插入数据操作

* 插入、更新、删除操作都可以使用`Exec`方法。

  * `Exec` 执行一次命令(包含查询、删除、更新、插入等), 返回的`Result`是对已执行的SQL命令的总结。

  * `func (db *DB) Exec(query string, args ...interface{}) (Result, error)` 方法


```go


func insertRow(name string, age int) {
	sqlStr := "insert into user(name,age) values (?,?)"
	ret, err := DB.Exec(sqlStr, name, age)
	if err != nil {
		fmt.Printf("Insert Row Error %v\n", err)
		return
	}
        var inID int64
	// 获取Last最后一次插入数据的ID
	inID, err = ret.LastInsertId()
	if err != nil {
		fmt.Printf("Get LastInsertId Error %v\n", err)
		return
	}
	fmt.Printf("Insert Success ID %d \n", inID)
}


func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	insertRow("小炒肉", 10)
	queryMultiRowToID(0)
}

```

```shell
# 输出结果
Insert Success ID 4 
ID: 1 Name: jicki  Age: 20 
ID: 2 Name: Jack  Age: 30 
ID: 3 Name: Vicki  Age: 30 
ID: 4 Name: 小炒肉  Age: 10 

```



> 更新数据操作


```go

func updateRowToAge(age, id int) {
	sqlStr := "update user set age = ? where id = ?"
	ret, err := DB.Exec(sqlStr, age, id)
	if err != nil {
		fmt.Printf("Update Failed Error %v\n", err)
		return
	}
        var n int64
	// RowsAffected 返回更新影响的行数
	n, err = ret.RowsAffected()
	if err != nil {
		fmt.Printf("Get RowAffected Failed Error %v\n", err)
		return
	}
	fmt.Printf("Update Success Affected Row: %d \n", n)
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	queryRowToID(1)
	updateRowToAge(11, 1)
	queryRowToID(1)
}

```

```shell
# 输出结果

ID: 1 Name: jicki  Age: 20 
Update Success Affected Row: 1 
ID: 1 Name: jicki  Age: 11 

```



> 删除数据操作



```go

func deleteRowToID(id int) {
	sqlStr := "delete from user where id = ?"
	ret, err := DB.Exec(sqlStr, id)
	if err != nil {
		fmt.Printf("Delete Failed Error %v\n", err)
		return
	}
	var n int64
	// RowsAffected 返回更新影响的行数
	n, err = ret.RowsAffected()
	if err != nil {
		fmt.Printf("Get RowAffected Failed Error %v\n", err)
		return
	}
	fmt.Printf("Delete Success Affected Row: %d \n", n)
}


func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	queryMultiRowToID(0)
	deleteRowToID(3)
	queryMultiRowToID(0)
}

```


```shell
# 输出结果
ID: 1 Name: jicki  Age: 11 
ID: 2 Name: Jack  Age: 30 
ID: 3 Name: Vicki  Age: 30 
ID: 4 Name: 小炒肉  Age: 10 
Delete Success Affected Row: 1 
ID: 1 Name: jicki  Age: 11 
ID: 2 Name: Jack  Age: 30 
ID: 4 Name: 小炒肉  Age: 10 

```



### Mysql 预处理


* 预处理

  * 普通`SQL` 语句执行过程:

    * 客户端对 `SQL` 语句进行占位符替换得到完整的`SQL` 语句。
    * 客户端发送完整`SQL` 语句到 MySQL 服务端。
    * MySQL 服务端执行完整的 `SQL` 语句并将结果返回给客户端。

  * 预处理执行过程:

    * 把`SQL` 语句分成两部分, 命令部分与数据部分。
    * 把命令部分发送给 MySQL 服务端, MySQL 服务端进行`SQL` 预处理。
    * 数据部分发送给 MySQL 服务端, MySQL 服务端对`SQL` 语句进行占位符替换。
    * MySQL 服务端执行完整的 `SQL` 语句并将结果返回给客户端。



* 预处理的优点:

  * 优化 MySQL 服务重复执行`SQL` 语句, 可提高 MySQL 服务性能, 一次编译多次执行, 节省编译成本。

  * 避免 `SQL` 注入的安全问题。 


#### Go 实现MySQL 预处理


* `database/sql` 包中使用 `Prepare` 方法实现预处理操作。


* `func (db *DB) Prepare(query string) (*Stmt, error)`


* `Prepare` 方法会先将 `SQL` 语句发送给 MySQL 服务端, 服务端返回一个状态用于后续的查询和操作。返回值可以同时执行多个查询和操作。 



> 预处理 查询例子

```go

type user struct {
	id   int
	name string
	age  int
}

func perPareQuery(id int) {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := DB.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("Prepare Failed Error %v\n", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(id)
	if err != nil {
		fmt.Printf("Prepare Query Error %v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("Perpare Scan Failed Error %v\n", err)
			return
		}
		fmt.Printf("ID: %d  Name: %s Age: %d \n", u.id, u.name, u.age)
	}
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	perPareQuery(0)
}

```


```shell
# 输出结果
ID: 1  Name: jicki Age: 11 
ID: 2  Name: Jack Age: 30 
ID: 4  Name: 小炒肉 Age: 10 
```



> 预处理 插入处理


```go

func perPareInsert(name string, age int) {
	sqlStr := "insert into user(name, age) values (?, ?)"
	stmt, err := DB.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("Prepare Failed Error %v\n", err)
		return
	}
	defer stmt.Close()
	result, err := stmt.Exec(name, age)
	if err != nil {
		fmt.Printf("Prepare Exec Error %v\n", err)
		return
	}
	InID, err := result.LastInsertId()
	if err != nil {
		fmt.Printf("Get LastInsertID Error %v \n", err)
		return
	}
	fmt.Printf("Insert ID: %d Success \n", InID)
}


func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	perPareQuery(0)
	perPareInsert("大炒肉", 20)
	perPareQuery(0)
}



```

```shell
# 输出结果

ID: 1  Name: jicki Age: 11 
ID: 2  Name: Jack Age: 30 
ID: 4  Name: 小炒肉 Age: 10 
ID: 5  Name: 小小肉 Age: 20 
Insert ID: 6 Success 
ID: 1  Name: jicki Age: 11 
ID: 2  Name: Jack Age: 30 
ID: 4  Name: 小炒肉 Age: 10 
ID: 5  Name: 小小肉 Age: 20 
ID: 6  Name: 大炒肉 Age: 20 

```



#### SQL 注入问题

* 我们任何时候都不应该自己拼接`SQL`语句。


```go

func sqlInject(name string) {
	sqlStr := fmt.Sprintf("select id,name,age from user where name='%s'", name)
	fmt.Println("SQL: ", sqlStr)
	var u user
	err := DB.QueryRow(sqlStr).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("QueryRow Falied Error %v\n", err)
		return
	}
	fmt.Printf("ID: %d  Name: %s  Age: %d \n", u.id, u.name, u.age)
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	sqlInject("xxxx' or 1=1#")
}

```

```shell
# 输出结果

SQL:  select id,name,age from user where name='xxxx' or 1=1#'
ID: 1  Name: jicki  Age: 11 

```




### Go 实现 MySQL 事务

* MySQL 只有使用 `Innodb` 数据引擎的数据库或表才支持事务。事务处理可以用来维护数据库的完整性, 保证多条联合 `SQL`语句要么全部执行, 要么全部不执行(回滚)。

> 什么是事务

* 事务: 一个最小的不可再分的工作单元, 通常一个事务对应一个完整的业务, 这个完整的业务需要执行多次(增Insert, 删Delete, 改Update)语句共同联合完成。




#### 事务的ACID

* 通常事务必须满足4个条件既满足(ACID)

  * `Atomictiy` 原子性、不可分割性
    * 一个事务中的所有操作, 要么全部完成, 要么全部不执行, 不会结束在中间环节。
    * 一个事务在执行过程中出现错误, 整个事务都会回滚(Rollback)到事务开始前的状态。

  * `Consistency` 一致性
    * 事务开始之前和事务结束以后, 数据的完整性没有被破坏。
    * 数据写入必须完全符合所规定的规则, 包含数据的精准度、串联性以及后续自发性地完成预定工作。

  * `Isolation` 隔离性、独立性
    * 数据库允许多个并发事务同时对数据进行读写和修改的能力。
    * 隔离性可以防止多个事务并发执行时由于交叉执行导致的数据不一致问题。
    * 事务隔离分为不同级别, 包括只读不提交(Read uncommitted)、读提交(Read committed)、可重复读(Repeatable Read)、串行化(Serializable)。

  * `Durability` 持久性
    * 事务处理结束后, 对数据的修改是永久性的, 故障不丢失。



#### Go 操作事务的方法



> 开始事务

```go
func (db *DB) Begin() (*Tx, error) 
```


> 提交事务

```go
func (db *DB) Commit() error

```

> 回滚事务

```go
func (tx *Tx) Rollback() error

```




#### 事务操作例子


```shell
# MySQL 表结果

ID: 1  Name: jicki Age: 11 
ID: 2  Name: Jack Age: 18 
ID: 4  Name: 小炒肉 Age: 10 
ID: 5  Name: 小小肉 Age: 20 
ID: 6  Name: 大炒肉 Age: 20 

```



```go

func transaction() {
	tx, err := DB.Begin()
	if err != nil {
		if tx != nil {
			err = tx.Rollback()
			if err != nil {
				fmt.Printf("Rollback Failed Error %v\n", err)
				return
			}
			fmt.Printf("Transaction Begin Error %v\n", err)
			return
		}
	}
	sqlStr := "update user set age = 18 where id = ?"
	rs1, err := tx.Exec(sqlStr, 2)
	if err != nil {
		err = tx.Rollback()
		fmt.Printf("rs1 Exec Failed Error %v\n", err)
		if err != nil {
			fmt.Printf("Rollback Failed Error %v\n", err)
			return
		}
	}
	rowID1, _ := rs1.RowsAffected()

	sqlStr = "update user set age = 40 where id = ?"
	rs2, err := tx.Exec(sqlStr, 3)
	if err != nil {
		err = tx.Rollback()
		fmt.Printf("rs2 Exec Failed Error %v\n", err)
		if err != nil {
			fmt.Printf("Rollback Failed Error %v\n", err)
			return
		}
	}
	rowID2, _ := rs2.RowsAffected()
	if rowID1 == 1 && rowID2 == 1 {
		fmt.Println("事务提交....")
		err = tx.Commit()
		if err != nil {
			err = tx.Rollback()
			fmt.Printf("Exec Failed Error %v\n", err)
			if err != nil {
				fmt.Printf("Rollback Failed Error %v\n", err)
				return
			}
		}
	} else {
		fmt.Println("事务回滚....")
		err = tx.Rollback()
		if err != nil {
			fmt.Printf("Rollback Failed Error %v\n", err)
			return
		}
	}
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Printf("initDB failed, err:%v \n", err)
		return
	}
	defer DB.Close()
	transaction()
}

```


```shell
# 输出结果
事务回滚....
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




### 初始化数据库


```go
package main

import (
	"fmt"
	"time"

	// 引入 mysql 驱动, 只需要用到 init() 方法
	_ "github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// 定义全局的 数据库对象 DB
var DB *sqlx.DB

func initDB() (err error) {
	dsn := "root:jicki@tcp(127.0.0.1:13306)/jicki?charset=utf8mb4&parseTime=True"

	// 这里 DB 赋值是给 上面定义的全局变量赋值, 不要使用 :=
	DB, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		return
	}
	// 连接存活时间,超时会被自动关闭
	DB.SetConnMaxLifetime(time.Second * 10)
	// 最大连接数
	DB.SetMaxOpenConns(100)
	// 连接池中空闲连接数
	DB.SetMaxIdleConns(50)
	return
}

func main() {
	if err := initDB(); err != nil {
		fmt.Printf("InitDB Failed Error %v\n", err)
		return
	}

}
```



### 插入数据库

```shell
/*
数据库 sqlx 表

CREATE TABLE `sqlx` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(64) NOT NULL,
  `age` int(6) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

*/
```

```go

// 创建一个结构体,并为 db 打上标签
type user struct {
	Id   int    `db:"id"`
	Name string `db:"name"`
	Age  int    `db:"age"`
}

func InsertSqlX(u user) {
        // (:后面是数据库字段), 最后传入整个 结构体
	sqlStr := "insert into sqlx values (:id, :name, :age)"
	result, err := DB.NamedExec(sqlStr, u)
	if err != nil {
		fmt.Printf("Insert NamedExec Failed Error %v\n", err)
		return
	}
	var TheID int64
	TheID, err = result.LastInsertId()
	if err != nil {
		fmt.Printf("InsertSqlx Get RowsAffected Falied Error %v\n", err)
		return
	}
	fmt.Printf("InsertSqlx Exec Success %d \n", TheID)
}

func main() {
	if err := initDB(); err != nil {
		fmt.Printf("InitDB Failed Error %v\n", err)
		return
	}
	s1 := user{
		Id:   1,
		Name: "小炒肉",
		Age:  20,
	}
	s2 := user{
		Id:   2,
		Name: "大炒肉",
		Age:  30,
	}
	InsertSqlX(s1)
	InsertSqlX(s2)
}

```


```shell
# 输出结果
InsertSqlx Exec Success 1 
InsertSqlx Exec Success 2 

```

### 查询数据

>  查询单条数据

```go
// 创建一个结构体,并为 db 打上标签
type User struct {
	Id   int    `db:"id"`
	Name string `db:"name"`
	Age  int    `db:"age"`
}

func queryRowToID(id int64) {
	sqlStr := "select id, name, age from sqlx where id = ?"
	var u User
	err := DB.Get(&u, sqlStr, id)
	if err != nil {
		fmt.Printf("QueryRowToID Get Failed Error %v\n", err)
		return
	}
	fmt.Printf("ID: %d  Name: %s  Age: %d \n", u.Id, u.Name, u.Age)
}

func main() {
	if err := initDB(); err != nil {
		fmt.Printf("InitDB Failed Error %v\n", err)
		return
	}
	queryRowToID(1)
}
```

```shell
# 输出结果
ID: 1  Name: 小炒肉  Age: 20 

```

> 查询多条数据

```go
// 创建一个结构体,并为 db 打上标签
type User struct {
        Id   int    `db:"id"`
        Name string `db:"name"`
        Age  int    `db:"age"`
}

func queryMultiRows(id int64) {
	sqlStr := "select id, name, age from sqlx where id > ?"
        // 查询的结果存储到 User结构体类型的切片 u 中
	var u []User
        // Select 方法用于查询多条数据
	err := DB.Select(&u, sqlStr, id)
	if err != nil {
		fmt.Printf("QueryMultiRows Select Failed Error %v\n", err)
		return
	}
	for _, v := range u {
		fmt.Printf("ID: %d  Name: %s  Age: %d \n", v.Id, v.Name, v.Age)
	}
}

func main() {
	if err := initDB(); err != nil {
		fmt.Printf("InitDB Failed Error %v\n", err)
		return
	}
	queryMultiRows(0)
}

```


```shell
# 输出结果
ID: 1  Name: 小炒肉  Age: 20 
ID: 2  Name: 大炒肉  Age: 30 
ID: 3  Name: 小小肉  Age: 18 

```


> 更新数据


```go

func updateRow(id, age int64) {
	sqlStr := "update sqlx set age = ? where id = ?"
	result, err := DB.Exec(sqlStr, age, id)
	if err != nil {
		fmt.Printf("UpdateRow Exec Failed Error %v\n", err)
		return
	}
	var n int64
	n, err = result.RowsAffected()
	if err != nil {
		fmt.Printf("updateRow Get RowsAffected Failed Error %v\n", err)
		return
	}
	fmt.Printf("update id: %d To age = %d  Success Rows: %d \n", id, age, n)
}

func main() {
	if err := initDB(); err != nil {
		fmt.Printf("InitDB Failed Error %v\n", err)
		return
	}
	queryMultiRows(0)
	updateRow(1, 22)
	queryMultiRows(0)
}

```


```shell
# 输出结果

ID: 1  Name: 小炒肉  Age: 20 
ID: 2  Name: 大炒肉  Age: 30 
ID: 3  Name: 小小肉  Age: 18 
update id: 1 To age = 22  Success Rows: 1 
ID: 1  Name: 小炒肉  Age: 22 
ID: 2  Name: 大炒肉  Age: 30 
ID: 3  Name: 小小肉  Age: 18 

```



> 删除数据

```go
func deleteRowToId(id int64) {
	sqlStr := "delete from sqlx where id = ?"
	result, err := DB.Exec(sqlStr, id)
	if err != nil {
		fmt.Printf("UpdateRow Exec Failed Error %v\n", err)
		return
	}
	var n int64
	n, err = result.RowsAffected()
	if err != nil {
		fmt.Printf("DeleteRow Get RowsAffected Failed Error %v\n", err)
		return
	}
	fmt.Printf("delete id: %d Success  Rows: %d \n", id, n)
}

func main() {
	if err := initDB(); err != nil {
		fmt.Printf("InitDB Failed Error %v\n", err)
		return
	}
	queryMultiRows(0)
	deleteRowToId(3)
	queryMultiRows(0)
}
```

```shell
# 输出结果

ID: 1  Name: 小炒肉  Age: 22 
ID: 2  Name: 大炒肉  Age: 30 
ID: 3  Name: 小小肉  Age: 18 
delete id: 3 Success  Rows: 1 
ID: 1  Name: 小炒肉  Age: 22 
ID: 2  Name: 大炒肉  Age: 30 

```

