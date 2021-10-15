# 一、DDL数据定义

## 1.1创建数据库

* 创建一个数据库

```
create database [IF NOT EXISTS] 数据库名 [LOCATION PATH];
``` 
* 字段解释：
  * [IF NOT EXISTS] ： 创建数据库之前先判断是否有同名的数据库已经存在
  * [LOCATION PATH] ： 指定数据库在HDFS上存放的位置

## 1.2 查询数据库

### 1.2.1 显示数据库
```
show databases;
show databases LIKE '***'; //过滤显示查询的数据库
```

### 1.2.2 查看数据库详情
```
desc database 数据库名称
desc database extended  数据库名称; //显示详细的信息
```

### 1.2.3 切换数据库
```
use 数据库名称;
```

## 1.3 修改数据库
用户可以使用ALTER DATABASE命令为某个数据库的DBPROPERTIES设置键-值对属性值，来描述这个数据库的属性信息。
* 注意：**数据库的其他元数据信息都是不可更改的，包括数据库名和数据库所在的目录位置**。


```
alter database 数据库名称 set dbproperties('属性名'='属性值');
```

## 1.4 删除数据库

删除空数据库
```
drop database 数据库名;
drop database if exists 数据库名; //如果删除的数据库不存在，最好采用 if exists判断数据库是否存在
drop database 数据库名 cascade; //如果数据库不为空，可以采用cascade命令，强制删除
```

## 1.5 创建表

建表语法：
```
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
```

字段说明：
* CREATE TABLE ： 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。
* EXTERNAL关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION）。
  * Hive创建内部表时，会将数据移动到数据仓库指向的路径；
  * 若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。
  * 在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。
* COMMENT：为表和列添加注释。
* PARTITIONED BY创建分区表。
* STORED AS ： 指定存储文件类型。
  * 常用的存储文件类型：SEQUENCEFILE（二进制序列文件）、TEXTFILE（文本）、RCFILE（列式存储格式文件）
  * 如果文件数据是纯文本，可以使用STORED AS TEXTFILE。
  * 如果数据需要压缩，使用 STORED AS SEQUENCEFILE。
* ROW FORMAT ： 用户可以自定义，用于序列化和反序列化的格式。
* LOCATION ：指定表在HDFS上的存储位置。
* LIKE ： 允许用户复制现有的表结构，但是不复制数据。

### 1.5.1 内部表（管理表）
默认创建的表都是所谓的管理表，有时也被称为内部表。因为这种表被Hive（或多或少地）控制着数据的生命周期。Hive默认情况下会将这些表的数据存储在由配置项hive.metastore.warehouse.dir(例如，/user/hive/warehouse)所定义的目录的子目录下。	当我们删除一个管理表时，Hive也会删除这个表中数据。管理表不适合和其他工具共享数据

### 1.5.2 外部表
Hive并非认为其完全拥有这份数据。删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。

* 通常情况下，表是廉价的，但是数据是宝贵的，所以，在实际使用中，一般创建的都是外部表。

### 1.5.3 管理表与外部表的互相转换
```
alter table student2 set tblproperties('EXTERNAL'='TRUE');//修改内部表student2为外部表
alter table student2 set tblproperties('EXTERNAL'='FALSE');//修改外部表student2为内部表
```

## 1.6 分区表
分区表实际上就是对应一个HDFS文件系统上的独立的文件夹，该文件夹下是该分区所有的数据文件。Hive中的分区就是分目录，把一个大的数据集根据业务需要分割成小的数据集。在查询时通过WHERE子句中的表达式选择查询所需要的指定的分区，这样的查询效率会提高很多。

### 1.6.1 分区表基本操作
```
1) 创建分区表
create table dept_partition(
deptno int, dname string, loc string
)
partitioned by (month string)
row format delimited fields terminated by '\t';
2) 加载数据到分区表
load data local inpath 数据path into table dept_partition partition(分区列名=分区列值);
3) 查询数据
select * from dept_partition where 分区列名1=分区列值1;
4) 多分区联合查询
select * from dept_partition where 分区列名2=分区列值2
union
select * from dept_partition where 分区列名3=分区列值3
union
select * from dept_partition where 分区列名4=分区列值4;
5) 增加分区
alter table dept_partition add partition(分区列名=分区列值);
alter table dept_partition add partition(分区列名=分区列值) partition(分区列名2=分区列值2);//同时增加多个分区。
6) 删除分区
alter table dept_partition drop partition (分区列名=分区列值);
alter table dept_partition drop partition(分区列名=分区列值) partition(分区列名2=分区列值2);//同时删除多个分区。
7) 查看分区表有多少分区
show partitions dept_partition;
8) 查看分区表结构
desc formatted dept_partition;
```

### 1.6.2 分区表注意事项
```
1) 创建二级分区表
reate table dept_partition2(
deptno int, dname string, loc string
)
partitioned by (分区列名=分区列值, 分区列名2=分区列值2)
row format delimited fields terminated by '\t';
2) 让分区表和数据产生关联的三种方式
msck repair table dept_partition2; //上传数据后修复
alter table dept_partition2 add partition(分区列名=分区列值, 分区列名2=分区列值2);//上传数据后添加分区
load data local inpath '/opt/module/datas/dept.txt' into table dept_partition2 partition(分区列名=分区列值, 分区列名2=分区列值2); //创建文件夹后load数据到分区
```

## 1.7 修改表
```
ALTER TABLE table_name RENAME TO new_table_name; //重命名表
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name] //更新列
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...) //增加和替换列
drop table table_name; //删除表
```
