## 一、Hadoop的HA

### 1. HA
* H(high)A(avilable)： 高可用，意味着必须有容错机制，不能因为集群故障导致不可用！
* HDFS： 满足高可用
  * NN：  一个集群只有一个，负责接受客户端请求！
  * DN：  一个集群可以启动N个
  * YARN： 满足高可用
    * RM：  一个集群只有一个，负责接受客户端请求！
    * NM：  一个集群可以启动N个

    > 实现hadoop的HA，必须保证在NN和RM故障时，采取容错机制，可以让集群继续使用！
		
### 2. 防止故障
核心： **避免NN和RM单点故障**

以HDFS的HA为例：
* ①NN启动多个进程，一旦当前正在提供服务的NN故障了，让其他的备用的NN继续顶上
* ②NN负责接受客户端的请求
  * 在接收客户端的写请求时，NN还负责记录用户上传文件的元数据，保证： 正在提供服务的NN，必须和备用的NN之中的元数据必须是一致的！
  * 元数据的同步：
    * ①在active的nn格式化后，将空白的fsimage文件拷贝到所有的nn的机器上
    * ②active的nn在启动后，将edits文件中的内容发送给Journalnode进程，standby状态的nn主动从Journalnode进程拷贝数据，保证元数据的同步
								  
* **注意：**  
  * Journalnode在设计时，采用paxos协议, Journalnode适合在奇数台机器上启动！在hadoop中，要求至少需要3个Journalnode进程;
  * 如果开启了hdfs的ha,不能再启动2nn;
  * 当启动了多个NN时，是否允许多个NN同时提供服务？
    > 不允许多个NN同时对外提供服务，因为如果多个NN同时对外提供服务，那么在同步元数据时，非常消耗性能，而且容易出错！所以在同一时刻，最多只能有一个NN作为主节点，对外提供服务！其余的NN，作为备用节点！*使用active状态来标记主节点，使用standby状态标记备用节点！*

### 3. HDFS HA的搭建步骤
1、配置
* fs.defaultFS=hdfs://hadoop101:9000 进行修改
* 在整个集群中需要启动N个NN，配置N个NN运行的主机和开放的端口！
* 配置Journalnode
		
2、启动
* 先启动Journalnode
* 格式化NN，精格式化后的fsimage文件同步到其他的NN
启动所有的NN，需要将其中之一转为active状态