# 一、Namenode的工作原理

---

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
  * NameNode负责集群中所有客户端的请求和所有DataNode的请求，在一个集群中，通常NameNode需要一个高配置来保证NameNode可以及时处理这些请求，一定NameNode无法及时处理请求，HDFS就已经瘫痪
