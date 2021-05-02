# 一、Namenode的工作原理

## 1、NameNode的作用
* 保存HDFS上所有文件的元数据
* 接收客户端的请求
* 接收DataNode上报的信息，给DataNode分配任务(维护副本数量)
* ### **1）元数据的存储**
  *  元数据存储在fsimage文件和edits文件中
     *  fsimage--元数据的快照文件
        *  fsimage文件是如何产生的：
           *  首先：在hadoop集群第一次格式化是，会创建NameNode的工作目录，并在这个目录中生成一个fsimage_000000000000文件
           *  其次：在NameNode启动时，NameNode会将所有的edits文件个fsimage文件加载到内存合并得到最新的元数据信息，后将元数据持久化到磁盘生成新的fsimage文件
           *  如果，hadoop集群启用了2nn，2nn也会辅助NameNode辅助合并元数据，并将合并后的元数据发送到NameNode
     *  edits--记录所有写操作的文件
        *  NameNode启动后，每次接收的写操作都会记录到edits文件中，edits文件每个一定的时间和大小滚动。
  * txid: 每次写操作命令，分解成若干步，每一个步骤都会有一个id，这个id就是txid。
  * NameNode负责集群中所有客户端的请求和所有DataNode的请求，在一个集群中，通常NameNode需要一个高配置来保证NameNode可以及时处理这些请求，一定NameNode无法及时处理请求，HDFS就已经瘫痪
* ### **2）NameNode元数据的组成**
  * （1）inodes：记录在fsimage文件或edits文件中
  * （2）blocklist：块的位置信息，每次DataNode启动后自动上报，无法从配置文件中获取。

## 2、NameNode的启动过程
* 1）先加载fsimage_0000000000xxx文件到内存
* 2）将fsimage文件xxx之后的edits文件加载到内存
* 3）合并生成最新的元数据，记录checkpoint，如果满足要求，执行saveNamespace操作，不满足等待满足条件后执行（saveNamespace必须在安全模式下进行）
* 4）自动进入安全模式，等待DataNode上报块的信息，满足条件后离开安全模式
---

# 二、HDFS集群的注册

## 1、VERSION文件
* 每次格式化NameNode，会产生一个VERSOION文件，负责记录NameNode的集群信息
* 每次格式化NameNode时，重新生成clusterID和blockpoolId（会被DataNode领取，生成一个同名目录，每次DataNode启动时，会将这个同名目录中的块上报给NameNode）

## 2、NameNode中的VERSION

```
#Sat May 01 11:00:35 CST 2021
namespaceID=1510717386
clusterID=CID-314ce333-4b1e-48a4-94e9-fd152d5f7a30
cTime=0
storageType=NAME_NODE
blockpoolID=BP-930909093-192.168.134.161-1617984597767
layoutVersion=-63
```

## 3、DataNode中的VERSIO

```
#Sat May 01 11:00:36 CST 2021
storageID=DS-2850aa81-35cd-4265-9811-fc5cdf76867a
clusterID=CID-314ce333-4b1e-48a4-94e9-fd152d5f7a30
cTime=0
datanodeUuid=7b49ff48-f0fa-444a-a0ce-3199e2b9132a
storageType=DATA_NODE
layoutVersion=-56
```

### *DataNode在第一次启动时，如果没有VERSION的信息，回想配置文件中配置的NameNode发起请求，生成VERSION，加入到集群中。*

---

# 三、安全模式

## 1、NameNode在启动时，当NameNode将所有的元数据加载完成后，等待DataNode上报块的信息。
* 当NameNode中所保存的所有块的最小副本数（默认1） / 块的总数 > 0.9999时，NameNode会离开安全模式。
* 在安全模式下，客户端只能进行有限的去操作，不能进行写操作。
