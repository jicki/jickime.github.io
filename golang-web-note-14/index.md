# 通讯协议 - TLV


# TLV

* 通信协议可以理解两个节点之间为了协同工作实现信息交换, 协商一定的规则和约定, 例如规定字节序, 各个字段类型, 使用什么压缩算法或加密算法等。常见的有`tcp`, `udp`,`http`, `sip`等常见协议。协议有流程规范和编码规范。流程如呼叫流程等信令流程, 编码规范规定所有信令和数据如何打包/解包。


## TLV 编码介绍

* `TLV` 是指由数据的类型`Tag`, 数据的长度`Length`, 数据的值`Value`组成的结构体, 几乎可以描任意数据类型, `TLV`的`Value`也可以是一个`TLV`结构, 正因为这种嵌套的特性, 可以让我们用来包装协议的实现。

|T|L|V|
|-|-|-|
|Tag|Length|Value|



* `Tag` - 描述`Value`的数据类型, `TLV`嵌套时可以用于描述消息的类型。

* `Length` - 描述`Value`的长度, `Value`部分所占字节的个数, 编码格式分两类: 定长方式`(DefiniteForm)`和不定长方式`(IndefiniteForm)`。

* `Value` - 描述数据的值, 由一个或多个值组成, 值可以是一个原始数据类型`(Primitive Data)`, 也可以是一个`TLV`结构`(Constructed Data)`

  * `Primitive Data` 编码 
 
|Tag|Length|Value|

  * `Constructed Data` 编码

|Tag|Length|`[T][L][TLV][T|L|V]`|


## TLV 解决黏包

* 解决 TCP 黏包问题
  * 一个完整的包,包含 `head` 和 `body`

|head|body|
|-|-|
|DataLen和id|Data数据|
|第一次Read获取DataLen|第二次Read 根据DataLen 偏移读取消息数据|


