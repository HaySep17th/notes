# MapReduce核心编程思想

---

## 一、概念

### 1、Job（作业）：一个MR程序被称为一个Job

### 2、MRAppMaster（MR任务的主节点）：
* 一个Job在运行时，回西安启动一个MRAppMaster进程，负责Job中执行状态的监控、容错、向ResourceManager申请资源以及提交Task等

### 3、Task（任务）：是一个进程，负责某项计算

### 4、Map阶段：是MR程序运行的第一个阶段
* 目的是将输入的数据进行切分，将一个大数据切分为若干小数据
* 切分后，每个部分被称为一片（split），每片数据都会交给一个Task进行计算
* Task负责的是Map阶段程序的计算，称为MapTask，在一个MR程序的Map阶段，会启动N个（取决于切片数量）MapTask，这些MapTask是并行运行的

### 5、Reduce阶段：是MR程序运行的第二个阶段
* 目的是将Map阶段每个MapTask计算后的结果进行合并汇总得到最终结果（##Reduce阶段是可选的！##）
* Task负责Reduce阶段的计算，被称为ReduceTask，一个Job可以通过设置启动N个ReduceTask，这些ReduceTask也是并行运行的（每个ReduceTask最终都会产生一个结果）

---

## 二、MapReduce中常用的组件

### 1、Mapper：map阶段核心的处理逻辑

### 2、Reducer：reduce阶段核心的处理逻辑

### 3、InputFormat：输入格式
* MR程序必须制定一个输入目录和一个输出目录，InputFormat代表输入目录中文件的格式。
  * 如果是普通文件，可以使用-FileInputFormat
  * 如果是SequenceFile（hadoop提供的一种文件格式），可以使用SequenceFileInputFormat
  * 如果处理的文件存放在数据库中，可以使用DBInputFormat

### 4、RecordReader：记录读取器
* 负责从输入格式中读取数据，并将读取到的每一条记录封装为key-value对象

### 5、OutPutFormat：输出格式
* opf表示的是MR处理后的结果将要以哪种格式写出
  * 将结果写出到一个普通文件中，可以使用FileOutputFormat！
  * 将结果写出到数据库中，可以使用DBOutPutFormat！
  * 将结果写出到SequeceFile中，可以使用SequnceFileOutputFormat

### 6、RecordWriter：记录写出器
* 负责将处理的结果以哪种格式写出到输出文件中

### 7、Partitioner：分区器
* 负责在Mapper将数据写出时，将keyout-valueout打上标记进行分区
* 目的是一个ReduceTask智慧处理一个分区的数据

---

## 三、在MR中数据的流程

### 1、InputFormat调用RecordReader，从输入目录的文件中，读取一组数据，封装为keyin-valuein对象
### 2、将封装好的key-value，交给Mapper.map()------>将处理的结果写出 keyout-valueou
### 3、ReduceTask启动Reducer，使用Reducer.reduce()处理Mapper写出的keyout-valueout，
### 4、OutPutFormat调用RecordWriter，将Reducer处理后的keyout-valueout写出到文件

---

## 四、MapReduce的运行流程概述

### **需求：统计/WordCount目录中每个文件的单词数量，a-p开头的单词存放到一个结果文件中，q-z开头的单词存放到一个结果文件中。

### 例如： 
* /WordCount/a.txt 200.mb
  * hello,hi,hadoop
  * hive,hadoop,hive,
  * zoo,spark,word
  * zoo,spark,word
  * ...
* /WordCount/b.txt 100.mb
  * hello,hi,hadoop
  * hive,hadoop,hive
  * zoo,spark,word
  * zoo,spark,word
  * ...

### 1、Map阶段(运行MapTask，将一个大的任务切分为若干小任务，处理输出阶段性的结果)
#### 1）切片
* 默认的切分策略是以文件为单位，以文件的块大小(128M)为片大小进行切片
  * split0:/WordCount/a.txt, 0-128M
  * split1:/WordCount/a.txt, 128M-200M
  * split2:/WordCount/b.txt, 0-100M

#### 2）运行MapTask，每个MT负责一个切片数据
* split0:/WordCount/a.txt, 0-128M ---》 MapTask1
* split1:/WordCount/a.txt, 128-200M ---》 MapTask2
* split2:/WordCount/a.txt, 0-100M ---》 MapTask3

#### 3）读取数据阶段
* 在MR中，所有数据必须封装为key-value
* MapTask1,2,3都会初始化一个InputFormat（默认TextInputFormat），每个InputFormat对象负责创建一个RecordReader(LineRecordReader)对象，RecordReader负责从每个切片的数据中读取数据，封装为key-value.
  * **LineRecordReader:** 将文件中的每一行封装为一个key（offset）-value(当前行的内容)
* 举例：
  * hello,hi,hadoop---->(0,hellomhi,hadoop)
  * hive,hadoop,hive---->(1,hive,hadoop,hive)
  * zoo,spark,word---->(2,zoo,spark,spark)
  * zoo,spark,word---->(2,zoo,spark,spark)

#### 4）进入Mapper的map()阶段
* map()是Map阶段的核心处理逻辑！ 单词统计! map()会循环调用，对输入的每个Key-value都进行处理！
  * 输入：(0,hello,hi,hadoop)
  * 输出：(hello,1),(hi,1),(hadoop,1)  
  * 输入：(20,hive,hadoop,hive)
  * 输出：(hive,1),(hadoop,1),(hive,1)  
  * 输入：(30,zoo,spark,wow)
  * 输出：(zoo,1),(spark,1),(wow,1)  
  * 输入：(40,zoo,spark,wow)
  * 输出：(zoo,1),(spark,1),(wow,1) 

#### 5）分区
* 根据需求，要启动两个ReduceTask，生成两个结果文件，需要将MapTask输出的结果进行分区
* 在Mapper输出后，调用Partitioner，对Mapper输出的key-value进行分区，分区后也会排序（默认字典顺序排序）
```
MapTask1:		   
0号区：  (hadoop,1)，(hadoop,1)，(hello,1),(hi,1),(hive,1),(hive,1)
1号区：  (spark,1),(spark,1),(wow,1) ，(wow,1),(zoo,1)(zoo,1)

MapTask2:		   
0号区：  。。。
1号区： ...


MapTask3:		   
0号区：   (hadoop,1),(hello,1),(hi,1),
1号区： (spark,1),(wow,1),(zoo,1)
```

### 2、Reduce阶段

#### 1）copy
* ReduceTask启动后，会启动shuffle线程，从MapTask中拷贝相应分区的数据
  * ReduceTask1: 只负责0号区
    * 将三个MapTask，生成的0号区数据全部拷贝到ReduceTask所在的机器：
		 (hadoop,1)，(hadoop,1)，(hello,1),(hi,1),(hive,1),(hive,1),(hadoop,1),(hello,1),(hi,1)
  * ReduceTask2: 只负责1号区
    * 将三个MapTask，生成的1号区数据全部拷贝到ReduceTask所在的机器:
		(spark,1),(spark,1),(wow,1) ，(wow,1),(zoo,1)(zoo,1),(spark,1),(wow,1),(zoo,1)

#### 2）sort
* ReduceTask1:	只负责0号区进行排序：
```
(hadoop,1)，(hadoop,1)，(hadoop,1),(hello,1),(hello,1),(hi,1),(hi,1),(hive,1),(hive,1)
```
* ReduceTask2: 只负责1号区进行排序：
```
(spark,1),(spark,1),(spark,1),(wow,1) ，(wow,1),(wow,1),(zoo,1),(zoo,1)(zoo,1)
```

#### 3）reduce
* ReduceTask1---->Reducer---->reduce(一次读入一组数据)

* 例如：
  ```
    输入： (hadoop,1)，(hadoop,1)，(hadoop,1)
	输出：   (hadoop,3)

	输入： (hello,1),(hello,1)
    输出：   (hello,2)
		
	输入： (hi,1),(hi,1)
	输出：  (hi,2)

	输入：(hive,1),(hive,1)
	输出： （hive,2）
    ```
* ReduceTask2---->Reducer----->reduce(一次读入一组数据)
* *同上*

#### 4）写出记录
* 调用OutPutFormat中的RecordWriter将Reducer输出的记录写出

#### 5）结果
* 在输出目录中，生成文件part-r-0000
```
hadoop  3
hello   2
hi  2
hive    2
```
* 在输出目录中，生成文件part-r-0001
```
spark   3
word    3
zoo 3 
```

---

## 五、总结

Map阶段(MapTask)：  切片(Split)----读取数据(Read)----交给Mapper处理(Map)----分区和排序(sort)
Reduce阶段(ReduceTask):  拷贝数据(copy)----排序(sort)----合并(reduce)----写出(write)

