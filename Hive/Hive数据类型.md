# 1.1、基本数据类型

| Hive数据类型 | Java数据类型 | 长度 | 例子 |
| :----: | :----: | :----: | :----: |
| TINYINT | byte | 1byte,有符号整数 | 10 |
| SMALINT | short | 2byte,有符号整数 | 20 |
| INT | int | 4byte,有符号整数 | 30 |
| BIGINT | long | 8byte,有符号整数 | 40 |
| BOOLEAN | boolean | 布尔类型，true/false | TRUE |
| FLOAT | float | 单精度浮点数 | 1.1 |
| DOUBLE | double | 双精度浮点数 | 3.1415926 |
| STRING | string | 字符，可指定字符集，可以使用单引号或双引号 | “Hello，World！” |
| TIMESTAMP |  | 时间类型 |  |
| BINARY |  | 字节类型 |  |

* 对于HIve的STIRING类型相当于数据库的VARCHAR类型，该类型是一个可变的字符串，不过不能声明其中最多能存储多少个字符。

# 1.2 集合数据类型

| 数据类型 | 描述 | 语法实例 |
| :----: | :----: | :----: |
| STRUCT | 和c语言中的struct类似，都可以通过“点”符号访问元素内容。 | struct() |
| MAP | MAP是一组键-值对元组集合，使用数组表示法可以访问数据。 | map() |
| ARRAY | 数组是一组具有相同类型和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，编号从零开始. | array() |

* Hive有三种复杂数据类型ARRAY、MAP 和 STRUCT。ARRAY和MAP与Java中的Array和Map类似，而STRUCT与C语言中的Struct类似，它封装了一个命名字段集合，复杂数据类型允许任意层次的嵌套

* 示例

```
表结构：
{
  "name" : "hjc",
  "friends" : ["aaa", "bbb"], //ARRAY
  "hobby" : {                 //MAP
    "ball" : "busketball",
    "run" : "long way of fun"
  }
  "address" : {               //STRUCT
    "street" : "xiangheerxiang NO.30",
    "city" : "jinan"
  }
}
HIVE建表语句：
create table test(
name string,
friends array<string>,
children map<string, int>,
address struct<street:string, city:string>
)
row format delimited fields terminated by ','
collection items terminated by '_'
map keys terminated by ':'
lines terminated by '\n';
```
* 字段解释：
  * row format delimited fields terminated by ',' -- 列分隔符
  * collection items terminated by '_' -- MAP和STRUCT的分隔符
  * map keys terminated by ':' -- MAP中key与VALUE的分隔符
  * lines terminated by '\n' -- 行分隔符（默认\n）

* 注意：
  * MAP和STRUCT的区别：
    * MAP中的KEY是可以改变的，STRUCT中的KEY是不可变的。

# 1.3 类型转换

Hive的原子数据类型是可以进行隐式转换的，类似于Java的类型转换，例如某表达式使用INT类型，TINYINT会自动转换为INT类型，但是Hive不会进行反向转化，例如，某表达式使用TINYINT类型，INT不会自动转换为TINYINT类型，它会返回错误，除非使用**CAST操作**。

## 1.3.1 隐式类型转换

* （1）任何整数类型都可以隐式地转换为一个范围更广的类型，如TINYINT可以转换成INT，INT可以转换成BIGINT。
* （2）所有整数类型、FLOAT和STRING类型都可以隐式地转换成DOUBLE。
* （3）TINYINT、SMALLINT、INT都可以转换为FLOAT。
* （4）BOOLEAN类型不可以转换为任何其它的类型。

## 1.3.2 使用CAST操作显示转换
例如CAST('5' AS INT)将把字符串'5' 转换成整数5；如果强制类型转换失败，如执行CAST('abc' AS INT)，表达式返回空值 NULL。
