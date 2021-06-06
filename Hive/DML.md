# DML数据操作

## 1、数据导入

### 1.1 向表中加载数据

语法：load data [local] inpath '/opt/module/datas/student.txt' [overwrite] into table student [partition (partcol1=val1,…)];
（1）load data:表示加载数据

（2）local:表示从本地加载数据到hive表；否则从HDFS加载数据到hive表

（3）inpath:表示加载数据的路径

（4）overwrite:表示覆盖表中已有数据，否则表示追加

（5）into table:表示加载到哪张表

（6）student:表示具体的表

（7）partition:表示上传到指定分区

### 1.2 通过查询向表中加载数据
语法：
* 基本插入语法：insert into table  table_name partition(xxx=xxx) values(x,x);
* 根据查询结果插入语法：insert overwrite table tablename partition(xxx=xxx) select id, name from tablename where xxx=xxx;
* 多插入模式：
  * from table_name
  * insert overwrite table table_name partition(xxx=xxx)
  * select id, name where xxx=xxx
  * insert overwrite table table_name partition(xxx=xxx)
  * select id, name where xxx=xxx;

### 1.3 查询语句中创建表并加载数据

语法：create table if not exists table_name as select id, name from othertable;

### 1.4 Import数据到指定Hive表中

语法： import table table_name partition(xxx=xxx) from 文件路径;
* 注意：先用export导出后，再将数据导入。

## 2、数据导出

### 2.1 insert导出

* 将查询的结果导出到本地
  *  insert overwrite local directory 本地路径 select * from tablename;

* 将查询的结果格式化导出到本地
```
insert overwrite local directory 本地路径
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' 
select * from tablename;
```
* 将查询的结果导出到HDFS上
```
insert overwrite directory HDFS路径
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' 
select * from tablename;
```

### 2.2 Hadoop命令导出到本地
语法：dfs -get hive表文件的位置 本地路径;

### 2.3 Hive Shell 命令导出
语法：hive -e 'select * from 数据库名.tablename;' > 保存路径;

### 2.4 Export导出到HDFS上
语法： export table 数据库名.tablename to 本地路径;

## 3.清空数据
语法：truncate table 表名;
**注意：Truncate只能删除管理表，不能删除外部表中数据。**







