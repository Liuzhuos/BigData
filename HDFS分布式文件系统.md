# HDFS分布式文件系统

## 1.HDFS概述

1. Hadoop分布式文件系统（HDFS）是一种旨在在商品硬件上运行的分布式文件系统。

2. HDFS具有高度的容错能力，旨在部署在低成本硬件上。
3. HDFS提供对应用程序数据的高吞吐量访问，并且适用于具有大数据集的应用程序
4. HDFS放宽了一些POSIX要求，以实现对文件系统数据的流式访问。
5. HDFS最初是作为Apache Nutch Web搜索引擎项目的基础结构而构建的。
6. HDFS是Apache Hadoop Core项目的一部分。

------

## 2.不适用场景：

1. **低时间延迟数据访问的应用，例如几十毫秒范围。**         

 	原因：HDFS是为高数据吞吐量应用优化的，这样就会造成以高时间延迟为代价。

2. **大量小文件 。**        

​	原因：NameNode启动时，将文件系统的元数据加载到内存，因此文件系统所能存储的文件总数受限于NameNode内存容量。根据经验，每个文件，目录和数据块的存储信息大约占150字节，如果一百万个文件，且每个文件占一个数据块，那至少需要300MB的内存空间，但是如果存储十亿个文件，那么需要的内存空间将是非常大的。

3. **多用户写入，任意修改文件。**         

​	原因：现在HDFS文件只有一个writer，而且写操作总是写在文件的末尾。也不支持在文件的任意位置进行修改。可能以后会支持，但相对比较低效。

------

## 3.流式数据访问

​	在数据集生成后，长时间在此数据集上进行各种分析。每次分析都将涉及该数据集的大部分数据甚至全部数据，因此**读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要**。与流数据访问对应的是随机数据访问，它要求定位、查询或修改数据的延迟较小，比较适合于创建数据后再多次读写的情况，传统关系型数据库很符合这一点。 

------

## 4.文件存储级别

Bit B KB MB GB TB PB EB ZB YB BB NB DB

------

## 5.HDFS架构包含三个部分

![HDFS架构](tupian\3.png)

1. **NameNode**用于**存储、生成文件系统的元数据**。运行一个实例。
   1. 负责管理分布式文件系统的**命名空间（Namespace）**，保存了两个核心的数据结构，即**FsImage**和**EditLog**。
      1. **FsImage**用于维护文件系统树以及文件树中所有的文件和文件夹的元数据。
      2. 操作日志文件**EditLog**中记录了所有针对文件的创建、删除、重命名等操作。
2. DataNode（真正干活的）：DataNode用于**存储实际的数据**，将自己管理的数据块上报给NameNode ，运行多个实例。
3. Block：目前128MB为一块，不足也按128存储。块的大小远远大于普通文件系统，可以最小化寻址开销
   1. 支持大规模文件
   2. 存储简化系统设计
   3. 适合数据备份
4. Client：支持业务访问HDFS，从NameNode ,DataNode获取数据返回给业务。多个实例，和业务一起运行。
   1. 严格来说，客户端不算是HDFS的一部分
   2. 客户端是一个库，包含HDFS文件系统接口
   3. 支持打开、读取、写入等操作
   4. 提供JavaAPI，作为客户端编程接口

|              **NameNode**               |                **DataNode**                 |
| :-------------------------------------: | :-----------------------------------------: |
|               存储元数据                |                存储文件内容                 |
|           元数据保存在内存中            |             文件内容保存在磁盘              |
| 保存文件、Block、DataNode之间的映射关系 | 维护了Block id 到DataNode本地文件的映射关系 |

## 6.通信协议

HDFS是一个部署在集群上的分布式文件系统，因此，很多数据需要通过网络进行传输。

- 所有的HDFS通信协议都是构建在**TCP/IP**协议基础之上的。
- 客户端通过一个**可配置的端口**向名称节点主动发起TCP连接，并使用客户端协议与名称节点进行交互。
- 名称节点和数据节点之间则使用**数据节点协议**进行交互。
- 客户端与数据节点的交互是通过**RPC**（Remote Procedure Call）来实现的。在设计上，名称节点**不会主动发起RPC**，而是响应来自客户端和数据节点的RPC请求。

## 7.HDFS高可用

![image-20240429191804793](tupian\2.png)

主要体现在利用zookeeper实现主备NameNode，以解决**单点NameNode**故障问题。

- ZooKeeper主要用来存储HA下的状态文件，主备信息。ZK个数建议3个及以上且为**奇数**个（容灾能力和偶数相同）。
- NameNode主备模式，主提供服务，备同步主元数据并作为主的热备。
- ZKFC(ZooKeeper Failover Controller)用于监控NameNode节点的主备状态。
- JN(JournalNode)用于存储Active NameNode生成的Editlog。
- Standby NameNode加载JN上Editlog，同步元数据。
- ZKFC控制NameNode主备仲裁ZKFC作为一个精简的仲裁代理，其利用zookeeper的分布式锁功能，实现主备仲裁，再通过命令通道，控制NameNode的主备状态。ZKFC与NN部署在一起，两者个数相同。

**元数据同步**

- 主NameNode对外提供服务。生成的Editlog同时写入本地和JN，同时更新主NameNode内存中的元数据。 
- 备NameNode监控到JN上Editlog变化时，加载Editlog进内存，生成新的与主NameNode一样的元数据。
- 元数据同步完成。主备的FSImage仍保存在各自的磁盘中，不发生交互。FSImage是内存中元数据定时写到本地磁盘的副本，也叫元数据镜像。

**元数据持久化**

![image-20240429192342764](tupian\8.png)

- EditLog:记录用户的操作日志，用以在FSImage的基础上生成新的文件系统镜像。
- FSImage:用以阶段性保存文件镜像。
- FSImage.ckpt:在内存中对fsimage文件和EditLog文件合并（merge）后产生新的fsimage，写到磁盘上，这个过程叫**checkpoint.**。备用NameNode加载完fsimage和EditLog文件后，会将merge后的结果同时**写到本地磁盘和NFS**。此时磁盘上有一份原始的fsimage文件和一份新生成的checkpoint文件：fsimage.ckpt.  而后将fsimage.ckpt改名为fsimage（**覆盖**原有的fsimage）。
- EditLog.new: NameNode**每隔1小时或Editlog满64MB**就触发合并,合并时,将数据传到Standby NameNode时,因数据读写不能同步进行,此时NameNode产生一个新的日志文件Editlog.new用来存放这段时间的操作日志。Standby NameNode合并成fsimage后回传给主NameNode替换掉原有fsimage,并将Editlog.new 命名为Editlog。

# 8.数据备份机制

**副本放置策略**

- 第一个副本：放置在上传文件的数据节点；如果是集群外提交，则随机挑选一台磁盘不太满、CPU不太忙的节点
- 第二个副本：放置在与第一个副本不同的机架的节点上
- 第三个副本：与第一个副本相同机架的其他节点上
- 更多副本：随机节点

**副本距离计算公式**

- Distance(Rack1/D1, Rack1/D1)=0，同一台服务器的距离为0。
- Distance(Rack1/D1, Rack1/D3)=2，同一机架不同的服务器距离为2。
- Distance(Rack1/D1, Rack2/D1)=4，不同机架的服务器距离为4。不同数据中心的节点距离为6。

**完整性保障**

- **重建失效数据盘的副本数据**
  - DataNode向NameNode周期上报失败时，NameNode发起副本重建动作以恢复丢失副本。
- **集群数据均衡** 磁盘使用率
  - HDFS架构设计了数据均衡机制，此机制保证数据在各个DataNode上分布是**平均**的。
- **元数据可靠性保证**
  - 采用**日志机制**操作元数据，同时元数据存放在主备NameNode上。
  - **快照机制**实现了文件系统常见的快照机制，保证数据误操作时，能及时恢复。fsimage
- **安全模式**
  - HDFS提供独有安全模式机制，在数据节点故障，硬盘故障时，能防止**故障扩散**。

## 9.读写流程

![image-20240429193733727](tupian\7.png)

1. 业务应用调用HDFS Client提供的API，请求写入文件。
2. HDFS Client联系NameNode，NameNode在元数据中创建文件节点。
3. 业务应用调用write API写入文件。
4. HDFS Client收到业务数据后，从NameNode获取到数据块编号、位置信息后，联系DataNode，并将需要写入数据的DataNode建立起流水线。完成后，客户端再通过自有协议写入数据到DataNode1，再由DataNode1复制到DataNode2, DataNode3。
5. 写完的数据，将返回确认信息给HDFS Client。
6. 所有数据确认完成后，业务调用HDFS Client关闭文件。
7. 业务调用close, flush后HDFS Client联系NameNode，确认数据写完成，NameNode持久化元数据。

![image-20240429193923298](tupian\6.png)

1. 业务应用调用HDFS Client提供的API打开文件。
2. HDFS Client联系NameNode，获取到文件信息（数据块、DataNode位置信息）。
3. 业务应用调用read API读取文件。
4. HDFS Client根据从NameNode获取到的信息，联系DataNode，获取相应的数据块。
5. (Client采用就近原则读取数据)。
6. HDFS Client会与多个DataNode通讯获取数据块。
7. 数据读取完成后，业务调用close关闭连接。

