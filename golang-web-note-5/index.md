# GORM MYSQL


# Go Web 编程

## GORM

* ORM（Object Relation Mapping）, 对象关系映射, 实际上就是对数据库的操作进行封装, 对上层开发人员屏蔽数据操作的细节, 开发人员看到的就是一个个对象, 大大简化了开发工作, 提高了生产效率

* O = Object 对象 (程序中的对象/实例 如: Struct 结构体)

* R = Relational 关系 (关系型数据库 如: Mysql)

* M = Mapping 映射

  * 结构体 --> 数据表
  * 结构体实例 --> 数据库行
  * 结构体字段 --> 数据库字段



* ORM 优缺点

  * 优点 - 提高开发效率, 不需要使用SQL语句。

  * 缺点 - 执行性能较差、灵活性较差、弱化了SQL能力。


### GORM 概览

1. 全功能 ORM (无限接近)

2. 关联 (Has One, Has Many, Belongs To, Many To Many, 多态)

3. 钩子 (在创建/保存/更新/删除/查找之前或之后)

4. 预加载

5. 事务

6. 复合主键

7. SQL 生成器

8. 数据库自动迁移

9. 自定义日志

10. 可扩展性, 可基于 GORM 回调编写插件

11. 所有功能都被测试覆盖

12. 开发者友好


### gorm 模块用法

```shell
go get -u github.com/jinzhu/gorm

```

* gorm to mysql


```go

import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/mysql"
）

var db *gorm.DB

func init() {
    var err error
    db, err = gorm.Open("mysql", "<user>:<password>/<database>?charset=utf8&parseTime=True&loc=Local")
    if err != nil {
        panic(err)
    }
    // 设置连接池
    db.DB().SetMaxIdleConns(10)
    db.DB().SetMaxOpenConns(100)
}

```



### GORM Model

* 在使用ORM工具时，通常我们需要在代码中定义模型（Models）与数据库中的数据表进行映射，在GORM中模型（Models）通常是正常定义的结构体、基本的go类型或它们的指针。 同时也支持`sql.Scanner`及`driver.Valuer`、接口`interfaces`。



* GORM 内置了一个`gorm.Model`结构体。`gorm.Model`是一个包含了`ID` 、 `CreatedAt` 、`UpdatedAt` 、 `DeletedAt`四个字段的Golang结构体。

```go
// gorm.Model 定义
type Model struct {
  ID        uint `gorm:"primary_key"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt *time.Time
}
```


* 嵌入到自己的结构体模型中

```go
// 定义 数据模型
type UserInfo struct {
	gorm.Model   // 内嵌 Model 模型
	Name         string
	Age          sql.NullInt64 //零值类型
	Birthday     *time.Time
	Email        string  `gorm:"type:varchar(100);unique_index"`
	Role         string  `gorm:"size:255"`        // 限制大小为最多255
	MemberNumber *string `gorm:"unique;not null"` // 设置会员号唯一,且不能为空。
	Num          int     `gorm:"AUTO_INCREMENT"`  // 设置字段 num 自增类型。
	Address      string  `gorm:"index:addr"`      // 给address字段创建名为addr的索引
	IgnoreMe     int     `gorm:"-"`               // 忽略本字段
}

```



* GORM 结构体标识 (struct tag)

|结构体标识(Tga)|描述|
|-|-|
|Column |指定列名|
|Type |指定列数据类型|
|Size |指定列大小, 默认值255|
|PRIMARY_KEY |将列指定为主键|
|UNIQUE |将列指定为唯一|
|DEFAULT |指定列默认值|
|PRECISION |指定列精度|
|NOT NULL |将列指定为非 NULL|
|AUTO_INCREMENT |指定列是否为自增类型|
|INDEX |创建具有或不带名称的索引, 如果多个索引同名则创建复合索引|
|UNIQUE_INDEX |和 INDEX 类似，只不过创建的是唯一索引|
|EMBEDDED |将结构设置为嵌入|
|EMBEDDED_PREFIX |设置嵌入结构的前缀|
|- |忽略此字段|



* GORM 创建表时 set 的标识

|创建表的标识|描述|
|-|-|
|MANY2MANY |指定连接表|
|FOREIGNKEY |设置外键|
|ASSOCIATION_FOREIGNKEY |设置关联外键|
|POLYMORPHIC |指定多态类型|
|POLYMORPHIC_VALUE |指定多态值|
|JOINTABLE_FOREIGNKEY |指定连接表的外键|
|ASSOCIATION_JOINTABLE_FOREIGNKEY |指定连接表的关联外键|
|SAVE_ASSOCIATIONS |是否自动完成 save 的相关操作|
|ASSOCIATION_AUTOUPDATE |是否自动完成 update 的相关操作|
|ASSOCIATION_AUTOCREATE |是否自动完成 create 的相关操作|
|ASSOCIATION_SAVE_REFERENCE |是否自动完成引用的 save 的相关操作|
|PRELOAD |是否自动完成预加载的相关操作|



* 主键、表名、列名的约定
 
  * 主键 (Primary Key) , GORM 默认会使用名为`ID` 这个字段名作为表的主键。

  * 修改默认的 主键名为 其他, 需要使用结构体 tag `grom:"primary_key"` 来约定。

```go

type User struct {
  ID   string // 名为`ID`的字段会默认作为表的主键
  Name string
}

// 使用`AnimalID`作为主键
type Animal struct {
  AnimalID int64 `gorm:"primary_key"`
  Name     string
  Age      int64
}

```

  
* 表名 (Table Name) 

  * GROM 创建数据表, 表名默认就是结构体名称的复数, 如: `User = users` , `UserInfo = user_infos`

  * 修改默认的 数据表名称。

```go

// 将 User 的表名设置为 `profiles`
func (User) TableName() string {
  return "profiles"
}

// 多个条件判断
func (u User) TableName() string {
  if u.Role == "admin" {
    return "admin_users"
  } else {
    return "users"
  }
}

// 禁用默认表名的复数形式，如果置为 `true，则 `User` 的默认表名是 `user`
db.SingularTable(true)

```


```go
// table 可以指定表名

// 使用User结构体创建名为`deleted_users`的表
db.Table("deleted_users").CreateTable(&User{})

```


```go
// 修改默认表的规则 (格式为job_表明)
gorm.DefaultTableNameHandler = func (db *gorm.DB, defaultTableName string) string  {
  return "job_" + defaultTableName;
}
```



* 数据列 名称 (Column Name)

  * 列名 是由 结构体字段的名称组合,一个字段多个单词组成 会使用 `_` 分割开。如: `MemberNumber` = `member_number`

  * 修改默认的 数据列名称, 可使用结构体`tag` 中的 `column` 指定列名。

```go
type Animal struct {
  AnimalId    int64     `gorm:"column:beast_id"`        
  Birthday    time.Time `gorm:"column:beast_day"` 
  Age         int64     `gorm:"column:beast_age"`
}

```


* 时间类型 (At)

  * `CreatedAt`字段, 该字段的值将会是初次创建记录的时间。

  * `UpdatedAt`字段，该字段的值将会是每次更新记录的时间。 

  * `DeletedAt`字段，调用Delete方法删除该记录时，将会设置`DeletedAt`字段为当前时间，而不是直接将记录从数据库中删除。



### GORM CRUD

* CRUD 增、删、改、查



#### 增


* 增加(create)


```go

// 定义 数据模型
type User struct {
	ID int64
	// 使用 tag default 设置默认值
	Name string `gorm:"default:'小炒肉'"`
	Age  int64
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 创建结构体实例
	u1 := User{
		Name: "小炒肉",
		Age:  20,
	}
	// 判断主键是否存在,可用于判断数据是否成功写入。
	db.NewRecord(&u1)
	// 创建(插入)数据
	db.Create(&u1)
}
```

* 注意: 使用 结构体 tag default 设置默认值的时候, 所有字段的零值, 比如`0`, `""`, `false`或者其它零值，都不会保存到数据库内，但会使用他们的默认值。如需要零值/空值, 可以使用: 


* `Name *string `gorm:"default:'小炒肉'"`` 

```go
// 定义 数据模型
type User struct {
	ID int64
	// 使用 tag default 设置默认值
	Name *string `gorm:"default:'小炒肉'"`
	Age  int64
}

func main(){

	// 创建结构体实例
	u1 := User{
		Name: new(string),
		Age:  20,
	}
	// 判断主键是否存在,可用于判断数据是否成功写入。
	db.NewRecord(&u1)
	// 创建(插入)数据
	db.Create(&u1)
}

```



* `Name sql.NullString `gorm:"default:'小炒肉'"``


```go
type User struct {
        ID int64
        // 使用 tag default 设置默认值
        Name sql.NullString `gorm:"default:'小炒肉'"`
        Age  int64
}

func main(){

        // 创建结构体实例
        u1 := User{
                Name: sql.NullString{"", true},
                Age:  20,
        }
        // 判断主键是否存在,可用于判断数据是否成功写入。
        db.NewRecord(&u1)
        // 创建(插入)数据
        db.Create(&u1)
}

```


#### 查询

* 查询 (First、Take、Last、Find)

```go
func main() {
	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	var user User

	// 定义一个 User结构体类型的切片,用于存储查询返回的多条数据
	var users []User

	// First 取第一条数据
	db.First(&user)
	fmt.Printf("DB First User: %v \n", user)

	// Take 随机取一条数据
	db.Take(&user)
	fmt.Printf("DB Take User: %v \n", user)

	// Last 根据主键 的顺序查询最后一条数据
	db.Last(&user)
	fmt.Printf("DB Last User: %v \n", user)

	// Find 查询所有的数据
	db.Find(&users)
	for _, user := range users {
		fmt.Printf("DB Find Users: %v \n", user)
	}

	// First 也可以查询指定的数据, 当主键为整型时可用
	db.Debug().First(&user, 2)
	fmt.Printf("DB First 2 User: %v \n", user)
}

```


* 条件选取 (Where)


```go
func main() {

	// Where 条件查询

	// 过滤查询单条的数据
	db.Where("name = ?", "大炒肉").First(&user)
	fmt.Printf("Where First: %v \n", user)

	// 过滤查询所有的数据
	db.Where("name = ?", "大炒肉").Find(&users)
	fmt.Printf("Where Find: %v \n", users)

	// 过滤查询所有的数据 <> 不等于
	db.Where("name <> ?", "炒肉").Find(&users)
	fmt.Printf("Where Find <> : %v \n", users)

	// 多个条件 IN 查询所有数据
	db.Where("name IN(?)", []string{"小炒肉", "大炒肉"}).Find(&users)
	fmt.Printf("Where Find IN : %v \n", users)

	// 模糊匹配 查询所有的数据
	db.Where("name LIKE ?", "%小%").Find(&users)
	fmt.Printf("Where Find LIKE : %v \n", users)

	// AND 匹配
	db.Where("name =? AND age = ?", "小炒肉", "20").Find(&users)
	fmt.Printf("Where Find AND : %v \n", users)

	// Time 时间匹配
	db.Where("updated_at > ?", "2020-03-02 06:41:44").Find(&users)
	fmt.Printf("Where Find Time : %v \n", users)

	// BETWEEN 区间时间
	db.Where("created_at BETWEEN ? AND ?", "2020-01-01 09:09:01", "2020-03-02 06:41:50").Find(&users)
	fmt.Printf("Where Find BETWEEN : %v \n", users)

	// Where Struct 查询条件进行过滤查询
	db.Where(&User{Name: "小炒肉", Age: 20}).First(&user)
	fmt.Printf("Where Struct : %v \n", user)

	// Where Map 查询条件进行过滤查询
	db.Where(map[string]interface{}{"name": "大炒肉", "age": 30}).Find(&users)
	fmt.Printf("Where Map : %v \n", users)

	// Where 主键切片 查询条件 进行过滤查询
	db.Where([]int64{1, 2}).Find(&users)
	fmt.Printf("Where 切片 : %v \n", users)
}
```

* 提示：当通过结构体进行查询时，GORM将会只通过非零值字段查询，这意味着如果你的字段值为`0`，`''`，`false`或者其他零值时，将不会被用于构建查询条件。


* 条件排除 (Not)

```go
func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	var user User

	// 定义一个 User结构体类型的切片,用于存储查询返回的多条数据
	var users []User

	// Not
	db.Not("name", "小小").First(&user)
	fmt.Printf("Not First : %v \n", user)

	// Not In
	db.Not("name", []string{"小炒肉", "大炒肉"}).Find(&users)
	fmt.Printf("Not In Find : %v \n", users)

	// Not In Slice 主键, 主键必须为 Int 类型
	db.Not([]int64{2}).First(&user)
	fmt.Printf("Not Slice First : %v \n", user)

	// Not Sql 写法
	db.Not("name = ? ", "小炒肉").First(&user)
	fmt.Printf("Not Sql First : %v \n", user)

	// Not Struct
	db.Not(&User{Name: "小炒肉"}).First(&user)
	fmt.Printf("Not Struct First : %v \n", user)

}

```


* OR (或)

```go
func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 定义一个 User结构体类型的切片,用于存储查询返回的多条数据
	var users []User

	// Or
	db.Where("name = ? ", "小炒肉").Or("name = ? ", "大炒肉").Find(&users)
	fmt.Printf("Or Find : %v \n", users)

	// Or Struct
	db.Where("name = ? ", "小炒肉").Or(User{Age: 30}).Find(&users)
	fmt.Printf("Or Struct Find : %v \n", users)

	// Or Map
	db.Where("name = ? ", "小炒肉").Or(map[string]interface{}{"age": 30}).Find(&users)
	fmt.Printf("Or Struct Find : %v \n", users)
}

```



* 内联操作


```go
func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	var user User

	// 定义一个 User结构体类型的切片,用于存储查询返回的多条数据
	var users []User

	// 内联条件 Inline (内嵌 where 条件)

	// 根据主键获取记录 (只适用于整形主键)
	db.First(&user, 2)
	fmt.Printf("Inline int First : %v \n", user)

	// 根据主键获取记录, 如果它是一个非整形主键
	db.First(&user, "id = ?", "string_primary_key")
	fmt.Printf("Inline string First : %v \n", user)

	// SQL 语句方式
	db.Find(&users, "name = ?", "小炒肉")
	fmt.Printf("Inline SQL : %v \n", users)

	db.Find(&users, "name <> ? AND age > ?", "小炒肉", 20)
	fmt.Printf("Inline SQL : %v \n", users)

	// Struct
	db.Find(&users, User{Age: 20})
	fmt.Printf("Inline Struct : %v \n", users)

	// Map
	db.Find(&users, map[string]interface{}{"age": 20})
	fmt.Printf("Inline Map : %v \n", users)
}

```


* FirstOrInit

  * 获取匹配的第一条记录，否则根据给定的条件初始化一个新的对象 (仅支持 `struct` 和 `map` 条件)

```go

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	var user User

	// FirstOrInit 仅支持 Struct 和 Map
	// Struct 方式 Not Found 会初始化一条对象到 user 这个实例中,不会写入数据库
	db.FirstOrInit(&user, User{Name: "中炒肉", Age: 25})
	fmt.Printf("FirstOrInit Struct Not Found %v \n", user)

	// Map 方式 Not Found 会初始化一条对象到 user 这个实例中,不会写入数据库
	db.FirstOrInit(&user, map[string]interface{}{"name": "中炒肉", "age": 25})
	fmt.Printf("FirstOrInit Map Not Found %v \n", user)
}

```


* Attrs 与 Assign

  * Attrs 如果记录未找到， 将使用参数初始化 `struct` 。

  * Assign 不管记录是否找到，都将参数赋值给 `struct` 。

```go

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	var user User

	// attrs 如果记录未找到， 将使用参数初始化 struct
	db.Where(User{Name: "not_found"}).Attrs(User{Age: 99}).FirstOrInit(&user)
	fmt.Printf("Attrs 未找到记录 : %v \n", user)

	db.Where(User{Name: "中炒肉"}).Attrs(User{Age: 99}).FirstOrInit(&user)
	fmt.Printf("Attrs 找到记录 : %v \n", user)

	// Assign 不管记录是否找到，都将参数赋值给 struct.
	db.Where(User{Name: "not_found"}).Assign(User{Age: 99}).FirstOrInit(&user)
	fmt.Printf("Assign 未找到记录 : %v \n", user)

	db.Where(User{Name: "中炒肉"}).Assign(User{Age: 99}).FirstOrInit(&user)
	fmt.Printf("Assign 找到记录 : %v \n", user)
}

```


* FirstOrCreate 
  
  * 获取匹配的第一条记录, 否则根据给定的条件创建一个新的数据库记录 (仅支持 `struct` 和 `map` 条件)

```go

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	var user User

	// FirstOrCreate 结合 Attrs 如果记录未找到， 将使用参数初始化 struct 并创建数据库记录
	db.Where(User{Name: "巨炒肉"}).Attrs(User{Age: 40}).FirstOrCreate(&user)
	fmt.Printf("FirstOrCreate Attrs : %v \n", user)

	// FirstOrCreate 结合 Assign 不管记录是否找到，都将参数赋值给 struct. 并更新数据库对应记录
	db.Where(User{Name: "大炒肉"}).Assign(User{Age: 35}).FirstOrCreate(&user)
	fmt.Printf("FirstOrCreate Assign : %v \n", user)

}
```



### 高级查询


* 子查询 (QueryExpr)


```go
# 一个官方例子

db.Where("amount > ?", DB.Table("orders").Select("AVG(amount)").Where("state = ?", "paid").QueryExpr()).Find(&orders)
// SELECT * FROM "orders"  WHERE "orders"."deleted_at" IS NULL AND (amount > (SELECT AVG(amount) FROM "orders"  WHERE (state = 'paid')));

```



* 选择字段 (Select)


```go
func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 查询 select
	// 定义一个User结构体类型的变量用于存储查询返回的数据
	//var user User

	var users []User

	// Select 选择字段
	db.Select("name,age").Find(&users)
	fmt.Printf("Select Find : %v \n", users)

	// Select Map
	db.Select([]string{"name", "age"}).Find(&users)
	fmt.Printf("Select Map Find : %v \n", users)

}
```


* 排序 (Order)

  * 定从数据库中检索出记录的顺序。设置第二个参数 `reorder` 为 `true` ，可以覆盖前面定义的排序条件。


```go
	// 排序 Order
	// desc: 降序
	db.Order("age desc, name").Find(&users)
	fmt.Printf("Order  : %v \n", users)

	// 多字段排序
	db.Order("age desc").Order("name").Find(&users)
	fmt.Printf("Order  : %v \n", users)
```



* 数量 (Limit)

  * 指定从数据库检索出的最大记录数。


```go

	// 限制数量 Limit
	db.Limit(3).Find(&users)
	fmt.Printf("Limit 3 : %v \n", users)

```



* 偏移 (Offset)

  * 指定开始返回记录前要跳过的记录数。

```go
	// 偏移 Offset
	db.Offset(3).Find(&users)
	fmt.Printf("Offset 3 : %v \n", users)
```


* 总数 (Count)

  * 获取记录的总数。

  * `Count` 必须是链式查询的最后一个操作 ，因为它会覆盖前面的 `SELECT`，但如果里面使用了 `count` 时不会覆盖。


```go
func main() {

	var users []User
	var count int

	// 总数 Count
	db.Find(&users).Count(&count)
	fmt.Printf("Count %d \n", count)
}

```



* 扫描 (Scan)

  * 扫描结果记录到一个 `Struct` 实例中。


```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name string `gorm:"default:'小炒肉'"`
	Age  int64  `gorm:"default:99"`
}

type Result struct {
	Name string
	Age  int64
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 定义一个 Result 结构体类型的实例 result
	var result Result

	// 定义一个 Result切片类型的结构体 实例 results
	var results []Result

	// Scan 扫描单个结果 保存到 result 实例中。
	db.Table("users").Select("name, age").Where("name = ?", "小炒肉").Scan(&result)
	fmt.Printf("Scan Result : %v \n", result)

	// Scan 扫描多个结构 保存到 results 实例中。
	db.Table("users").Select("name, age").Where("id > ?", 0).Scan(&results)
	fmt.Printf("Scan Results : %v \n", results)

	// 使用SQL 语句
	db.Raw("SELECT name, age FROM users WHERE age > ?", 30).Scan(&results)
	fmt.Printf("Scan Results : %v \n", results)
}

```


* 立即执行方法 (Immediate Methods) 

  * 立即执行方法是指那些会立即执行`SQL`语句并发送到数据库的方法, 如: `Create`、`First`、`Find`、`Take`、`Save`、`Update`、`Delete`、`Scan`、`Row` 等.

  * 多个立即执行方法，后一个立即执行方法会复用前面那个立刻执行方法的条件( 不包含内联条件 )。


```go
func main() {
	var user User
        var count int64

	// 立即执行方法
	db.Debug().Select("name").Where("name = ?", "小炒肉").First(&user)
       // 执行语句: SELECT name FROM `users`  WHERE `users`.`deleted_at` IS NULL AND ((name = '小炒
肉')) ORDER BY `users`.`id` ASC LIMIT 1


	// 多个立即执行条件, 后一个会复用前一个的条件. ( 内联条件除外 )
	db.Debug().Where("name LIKE ?", "%炒肉").Find(&users, "id IN (?)", []int64{2, 3, 4}).Count(&count)
	fmt.Printf("Users : %v \n Count : %d \n", users, count)

	// 实际语句为以下两条:
	// SELECT * FROM `users`  WHERE `users`.`deleted_at` IS NULL AND ((name LIKE '%炒肉') AND (id IN (2,3,4)))
	// SELECT count(*) FROM `users`  WHERE `users`.`deleted_at` IS NULL AND ((name LIKE '%炒肉'))
	// Count 会复用 Where 后面的条件,  Find 里面的是 内联条件,所以不会复用 Find 内的。


}

```



* 范围 (Scopes)

  * Scope 是建立在链式操作基础上的方法。

  * 可以让代码更加通用,逻辑更清晰。

```go

package main

import (
	"fmt"

	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"
)

// 定义一个全局的 DB 实例
var DB *gorm.DB

// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name string `gorm:"default:'小炒肉'"`
	Age  int64  `gorm:"default:99"`
}

type Result struct {
	Name string
	Age  int64
}

// 查询年龄大于多少的用户
func AgeThan(age int64) func(db *gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		return db.Where("age > ?", age)
	}
}

// 查询返回指定用户
func NameThan(user []string) func(db *gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		return db.Where("name IN (?)", user)
	}
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	DB = db

	var users []User

	// Scopes 将多个条件合并起来操作
	DB.Scopes(NameThan([]string{"小炒肉", "大炒肉"}), AgeThan(19)).Find(&users)
	fmt.Println(users)
}

```


#### 更新

* 更新所有字段 ( Save )

```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 定义一个 User结构体的实例 user
	var user User

	// 查询单条记录,保存到 user 实例中
	db.First(&user)

	// 更新 user 实例中的数据
	user.Name = "炒肉"
	user.Age = 21

	// 利用 save 更新到数据库中, Save 会更新所有字段的数据
	db.Debug().Save(&user)
        // UPDATE `users` SET `created_at` = '2020-03-02 06:41:45', `updated_at` = '2020-03-04 19:04:53', `deleted_at` = NULL, `name` = '炒肉', `age` = 21, `active= false  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1  
	fmt.Printf("Save Age : %v \n", user)
}

```


* 更新指定字段 ( Update 、Updates )

```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	var user User

	// 查询第一条数据
	db.First(&user)

	// update 更新指定字段
	db.Debug().Model(&user).Update("name", "小炒肉")
	// UPDATE `users` SET `name` = '小炒肉', `updated_at` = '2020-03-04 19:28:27'  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1 

	// 根据指定条件 更新字段
	db.Debug().Model(&user).Where("age = ?", 21).Update("age", 20)

	// 使用 Map 更新多个字段
	db.Debug().Model(&user).Updates(map[string]interface{}{"name":"小小炒肉","age": 21,"active":true})

	// 使用Struct 更新多个字段, 只会更新 非零 值的字段, 所以 false 不会被更新
	db.Debug().Model(&user).Updates(User{Name:"小炒肉",Age:20,Active:false})

}

```


* 更新选定或忽略的字段

  * 选定 (Select)

  * 忽略 (Omit)

```go

// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})


	var user User

	db.First(&user)

	// 更新 选定或者忽略 的字段

	// 只更新选定的字段
	db.Debug().Model(&user).Select("name").Updates(map[string]interface{}{"name":"小小炒肉","age":30, "active":false})
	// UPDATE `users` SET `name` = '小小炒肉', `updated_at` = '2020-03-11 15:21:42'  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1


	// 忽略选定的字段更新其他字段
	db.Debug().Model(&user).Omit("age").Updates(map[string]interface{}{"name":"小炒肉","age":20, "active":true})
	// UPDATE `users` SET `active` = true, `name` = '小炒肉', `updated_at` = '2020-03-11 15:22:28'  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1 
}

```


* 不触发Hook更新


```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	var user User

	db.First(&user)

	// UpdateColumn 更新指定单个字段 并且不触发hook,既不更新(BeforeUpdate, AfterUpdate)方法
	db.Debug().Model(&user).UpdateColumn("name","小小炒肉")
	// UPDATE `users` SET `name` = '小小炒肉'  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1


	// UpdateColumns 更新多个字段 并且不触发hook,既不更新(BeforeUpdate, AfterUpdate)方法
	db.Debug().Model(&user).UpdateColumns(map[string]interface{}{"name":"小炒肉","age":20})
	// UPDATE `users` SET `age` = 20, `name` = '小炒肉'  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1  

}
```



* 批量更新

  * 批量更新字段的时候`Hook` 也不会执行。

```go

// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	// 批量更新 map[string]interface{} , map使用 Table指定 数据库名
	db.Debug().Table("users").Where("id IN (?)", []int{1,2,3}).Updates(map[string]interface{}{"age":22})
	// UPDATE `users` SET `age` = 22  WHERE (id IN (1,2,3))

	// 批量更新 struct 只能更新非零字段 , struct 使用 Model指定数据库对应的结构体名
	db.Debug().Model(User{}).Updates(User{Name:"小炒肉", Age:20})
	// UPDATE `users` SET `age` = 20, `name` = '小炒肉', `updated_at` = '2020-03-11 16:14:06'  WHERE `users`.`deleted_at` IS NULL

}

```




* 利用 SQL 表达式计算和更新数据

  * `gorm.Expr`


```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	var users []User
	db.Find(&users)

	// 使用 expr 的 sql 语句 将所有 age 都 +2
	db.Debug().Model(&users).Update("age", gorm.Expr("age + ?",2))
	// UPDATE `users` SET `age` = age + 2, `updated_at` = '2020-03-11 17:28:09'  WHERE `users`.`deleted_at` IS NULL


	// 使用 map 将 所有 age 都 - 2
	db.Debug().Model(&users).Updates(map[string]interface{}{"age":gorm.Expr("age - ?",2)})
	// UPDATE `users` SET `age` = age - 2, `updated_at` = '2020-03-11 17:28:09'  WHERE `users`.`deleted_at` IS NULL


	// 使用 Where 条件 加 grom.expr
	db.Debug().Model(&users).Where("age < 30").Update("age",gorm.Expr("age - ?",1))
	// UPDATE `users` SET `age` = age - 1, `updated_at` = '2020-03-11 17:28:10'  WHERE `users`.`deleted_at` IS NULL AND ((age < 30))

}
```



#### 删除

* GORM 删除记录时, 请必须指定主键的值, GORM是根据指定主键删除记录的, 如果没有指定主键的值, 那么GORM会删除Model指定的所有的数据。

* GORM 删除记录 都是进行软删除, 既将 `deleted_at` 更新为删除的时间, 实际数据库的数据并没有被删除, 但是 GORM 中查询是没有的。

```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	var users []User
	db.Find(&users)

	fmt.Printf("User DB : %v \n",users)

	// 删除, 删除都是软删除
	var user = User{}
	var user2 = User{}

	// 必须指定主键的 ID
	user.ID = 1
	db.Debug().Delete(&user)
	// UPDATE `users` SET `deleted_at`='2020-03-13 12:40:28'  WHERE `users`.`deleted_at` IS NULL AND `users`.`id` = 1

	// 如果未指定主键ID, 会删除所有的记录
	db.Debug().Delete(&user2)
	// UPDATE `users` SET `deleted_at`='2020-03-13 13:01:48'  WHERE `users`.`deleted_at` IS NULL
}
```


* 条件删除 (批量操作)


```go
// 定义 数据模型
type User struct {
	gorm.Model
	// 使用 tag default 设置默认值
	Name   string `gorm:"default:'小炒肉'"`
	Age    int64  `gorm:"default:99"`
	Active bool
}

func main() {
	dsn := "jicki:jicki123@tcp(127.0.0.1:3306)/jicki?charset=utf8mb4&parseTime=true"

	db, err := gorm.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}

	defer db.Close()

	// 创建 结构体User 对应的数据表
	db.AutoMigrate(&User{})

	var users []User

	db.Find(&users)

	fmt.Printf("User DB : %v \n",users)

	db.Debug().Where("name = ?","巨炒肉").Delete(User{})
	// UPDATE `users` SET `deleted_at`='2020-03-13 17:52:55'  WHERE `users`.`deleted_at` IS NULL AND ((name = '巨炒肉'))
}

	db.Debug().Delete(User{},"name = ?", "大炒肉")
	// UPDATE `users` SET `deleted_at`='2020-03-13 17:55:41'  WHERE `users`.`deleted_at` IS NULL AND ((name = '大炒肉'))


	// 查询所有记录,包括被软删除的
	db.Debug().Unscoped().Where("name LIKE ?","%炒肉").Find(&users)
	fmt.Printf("Unscoped DB : %v \n",users)

	// 物理删除
	db.Debug().Unscoped().Delete(User{},"name = ?", "巨炒肉")
	//  DELETE FROM `users`  WHERE (name = '巨炒肉')
```




