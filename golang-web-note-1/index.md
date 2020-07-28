# MVC 与 CLD 框架


# Go Web 编程

> 设计一个程序的结构，有一门专门的学问，叫做"架构模式"（architectural pattern），属于编程的方法论。

## MVC 框架

* MVC模式就是架构模式的一种。

* MVC - 这个模式认为，程序不论简单或复杂，从结构上看，都可以分成三层。

  * Model（模型）

    * Model 是Web应用中的`最底层` 用于处理数据逻辑的部分，包括Service层和Dao层。
    
    * Service层用于和数据库联动，放置业务逻辑代码，处理数据库的增删改查。
     
    * Dao层用于放各种接口，以备调用。

  * View（视图）

    * View 是Web应用中的`第一层` 用于处理响应给客户的页面的部分，例如我们写的html静态页面，jsp动态页面，这些最终响应给浏览器的页面都是视图, 通常视图是依据模型数据来创建的。

  * Controller（控制）
  
    * Controller 在Web应用中的`中间一层`，简而言之，就是Servlet。（实际上一个方法就相当于一个对应的Servlet）。

* 这三层是紧密联系在一起的，但又是互相独立的，每一层内部的变化不影响其他层。每一层都对外提供接口（Interface），供上面一层调用。这样一来，就实现了 模块化，修改外观或者变更数据都不用修改其他层，大大方便了维护和升级。

{{< mermaid >}}
graph TD;
  id1(浏览器)
  id2((Controller))
  id3((View))
  id4((Model))
  id5(DataBase)
  id1--Request-->id2;
  id3--response-->id1;
  id2--数据驱动-->id3; 
  id2--数据交互-->id4;
  id4--数据操作-->id5;
  id5--数据操作-->id4;
  id4--数据交互-->id2;
{{< /mermaid >}}


### Web经典三层架构

* 表现层，UI，User Interface：

  * 主要接受用户的请求和把相应的页面响应给用户浏览器, 页面 对应MVC中的视图（View）, 逻辑 对应MVC中的控制器（Controller），即Servlet服务器。

* 业务逻辑层，BLL，Business Logic Layer:

  * 对应MVC中模型（Model）中的Service层，与数据库联动处理增删改查。

* 数据访问层/持久层，DAL，Data Access Layer:

  * 对应MVC中模型（Model）中的Dao层，提供接口支持。



## CLD 框架

* `Controller` 层

  * 服务的入口, 负责处理路由、参数校检、请求转发。

* `Logic` 层

  * 逻辑(服务)层, 负责处理业务逻辑。

* `Dao` 层

  * 负责数据与存储相关的服务。


{{< mermaid >}}
graph LR;
    A[前端Vue 等] -->| B(LB Nginx 等)
    B -->| C(HTTP、Thrift、gRPC 协议等)
    C -->| D(Controller)
    D -->| E(Logic)
    E -->| F(Dao)
{{< /mermaid >}}

---

