**Hadoop2.9.0 主要由HDFS，Yarn和MapReduce三部分构成**

# 一览表
|  | NameNode | JournalNode | DataNode | Zookeeper | ZKFC | ResourceManager | NodeManager |
|--|--|--|--|--|--|--|--|
| itaojin101 | √ | √ |  |  | √ | √ |  |
| itaojin102 | √ | √ |  |  | √ | √ |  |
| itaojin103 |  | √ | √ |  |  |  | √ |
| itaojin106 |  |  | √ | √ |  |  | √ |
| itaojin107 |  |  | √ | √ |  |  | √ |
| itaojin105 |  |  |  | √ |  |  |  |

# HDFS

## 几种HDFS集群

### 单机环境
默认情况下，Hadoop被配置为以非分布式模式运行，作为单个Java进程。这对于调试非常有用。

搭建教程：[http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html#Standalone_Operation](http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html#Standalone_Operation)

### 伪分布式集群
Hadoop也可以在一个单节点上运行，在一个伪分布模式下，每个Hadoop守护进程都在一个单独的Java进程中运行。

搭建教程：[http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html#Standalone_Operation](http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/SingleCluster.html#Standalone_Operation)

### 分布式集群
在分布式模式下，Hadoop的每个节点可以运行在不同的服务器上，真正的实现了可扩展的分布式集群。
搭建教程：
[http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/ClusterSetup.html](http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-common/ClusterSetup.html)
[https://blog.csdn.net/a237428367/article/details/50462858](https://blog.csdn.net/a237428367/article/details/50462858)

**分布式集群又分为好几种模式：**

#### 含SecondaryNameNode节点
在0.x系列的Hadoop版本中，分布式集群中含有SecondaryNameNode节点。

有个让人疑惑的概念：Secondary NameNode，也叫辅助namenode。从命名看，好像是第二个namenode，用于备份主namenode，在主namenode失败后启动。其实不然，这是一个误区。**SecondaryNameNode的重要作用是定期通过编辑日志文件合并命名空间镜像，以防止编辑日志文件过大**。

#### 含CheckpointNode和BackupNode
在2.x系列的Hadoop产品中，包含这两个节点，且完全可以取代0.x系列遗留下来的SecondaryNameNode，当然，可以选择依然使用后者。

CheckpointNode和SecondaryNameNode的作用以及配置完全相同。

BackupNode提供了一个真正意义上的备用节点。它在内存中维护了一份从NameNode同步过来的fsimage，同时它还从namenode接收edits文件的日志流，并把它们持久化硬盘。BackupNode在内存中维护与NameNode一样的Matadata数据。

#### HA (High Availability) 高可用集群
由于NameNode的在集群中充当着大脑中枢的作用，所以一旦宕机整个集群随即瘫痪，HA机制应运而生，HA便是解决以上问题的一种机制。在2.x系列的Hadoop产品中得到支持，但是只能搭建2个NameNode。

#### 多个NameNode的HA高可用集群
在3.x系列的Hadoop产品中，NameNode已经支持部署大于2个的情况。

## 构成

HDFS大致由NameNode, SecondaryNameNode, CheckpointNode, BackupNode, DataNode, JournalNode, ZKFC等节点构成。

### NameNode

HDFS集群有两类节点以管理者和工作者的工作模式运行，namenode就是其中的管理者。它管理着文件系统的命名空间，维护着文件系统树及整棵树的所有文件和目录。这些信息以两个文件的形式保存于内存或者磁盘，这两个文件是：命名空间镜像文件fsimage和编辑日志文件edit logs ，同时namenode也记录着每个文件中各个块所在的数据节点信息。

**namenode对元数据的操作过程**
![è¿™é‡Œå†™å›¾ç‰‡æè¿°](https://img-blog.csdn.net/20170503091946427?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWFuZ2pqdWFu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
图中有两个文件：  
（1）fsimage:文件系统映射文件，也是元数据的镜像文件（磁盘中），存储某段时间namenode内存元数据信息。  
（2）edits log:操作日志文件。  
这种工作方式的特点：  
（1）namenode始终在内存中存储元数据（metedata）,使得“读操作”更加快、  
（2）有“写请求”时，向edits文件写入日志，成功返回后才修改内存，并向客户端返回。  
（3）fsimage文件为metedata的镜像，不会随时同步，与edits合并生成新的fsimage。

从以上特点可以知道，edits文件会在集群运行的过程中不断增多，占用更多的存储空间，虽然有合并，但是只有在namenode重启时才会进行。并且在实际工作环境很少重启namenode，  
这就带来了一下问题：  
（1）edits文件不断增大，如何存储和管理？  
（2）因为需要合并大量的edits文件生成fsimage，导致namenode重启时间过长。  
（3）一旦namenode宕机，用于恢复的fsiamge数据很旧，会造成大量数据的丢失。

### SecondaryNameNode
上述问题的解决方案就是运行辅助namenode–Secondary NameNode，为主namenode内存中的文件系统元数据创建检查点，Secondary NameNode所做的不过是在文件系统中设置一个检查点来帮助NameNode更好的工作。它不是要取代掉NameNode也不是NameNode的备份，  
SecondaryNameNode有两个作用，一是镜像备份，二是日志与镜像的定期合并。两个过程同时进行，称为checkpoint（检查点）。  
镜像备份的作用：备份fsimage(fsimage是元数据发送检查点时写入文件)；  
日志与镜像的定期合并的作用：将Namenode中edits日志和fsimage合并,防止如果Namenode节点故障，namenode下次启动的时候，会把fsimage加载到内存中，应用edits log,edits log往往很大，导致操作往往很耗时。**（这也是namenode容错的一套机制）**

**Secondary NameNode创建检查点过程**

![enter image description here](https://img-blog.csdn.net/20170503103823737?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWFuZ2pqdWFu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**Secondarynamenode工作过程**  
（1）SecondaryNameNode通知NameNode准备提交edits文件，此时主节点将新的写操作数据记录到一个新的文件edits.new中。  
（2）SecondaryNameNode通过HTTP GET方式获取NameNode的fsimage与edits文件（在SecondaryNameNode的current同级目录下可见到 temp.check-point或者previous-checkpoint目录，这些目录中存储着从namenode拷贝来的镜像文件）。  
（3）SecondaryNameNode开始合并获取的上述两个文件，产生一个新的fsimage文件fsimage.ckpt。  
（4）SecondaryNameNode用HTTP POST方式发送fsimage.ckpt至NameNode。  
（5）NameNode将fsimage.ckpt与edits.new文件分别重命名为fsimage与edits，然后更新fstime，整个checkpoint过程到此结束。  
SecondaryNameNode备份由三个参数控制fs.checkpoint.period控制周期（以秒为单位，默认3600秒），fs.checkpoint.size控制日志文件超过多少大小时合并（以字节为单位，默认64M）， dfs.http.address表示http地址，这个参数在SecondaryNameNode为单独节点时需要设置。

**从工作过程可以看出，SecondaryNameNode的重要作用是定期通过编辑日志文件合并命名空间镜像，以防止编辑日志文件过大。SecondaryNameNode一般要在另一台机器上运行，因为它需要占用大量的CPU时间与namenode相同容量的内存才可以进行合并操作。它会保存合并后的命名空间镜像的副本，并在namenode发生故障时启用。**

### CheckpointNode
CheckpointNode和SecondaryNameNode的作用以及配置完全相同。  
**启动命令:**`hdfs namenode -checkpoint`

配置文件：**core-site.xml**  
* fs.checkpoint.period
* fs.checkpoint,size
* fs.checkpoint.dir
* fs.checkpoint.edits.dir

### BackupNode
提供了一个真正意义上的备用节点。  
BackupNode在内存中维护了一份从NameNode同步过来的fsimage，同时它还从namenode接收edits文件的日志流，并把它们持久化硬盘。  
BackupNode在内存中维护与NameNode一样的Matadata数据。  
**启动命令:**`hdfs namenode -backup`

配置文件：**hdfs-site.xml**  
* dfs.backup.address
* dfs.backup.http.address

### DataNode
提供真实文件数据的存储服务，
* Block：最基本的存储单位，默认Block大小是128MB。
* Replication：多复本。默认是三个。（hdfs-site.xml的dfs.replication属性）

### JournalNode
两个NameNode为了数据同步，会通过一组称作JournalNodes的独立进程进行相互通信。当active状态的NameNode的命名空间有任何修改时，会告知大部分的JournalNodes进程。standby状态的NameNode有能力读取JNs中的变更信息，并且一直监控edit log的变化，把变化应用于自己的命名空间。standby可以确保在集群出错时，命名空间状态已经完全同步了。

### ZKFailoverController (ZKFC)
当Active的NameNode宕机后，ZKFC会使Standby状态的NameNode自动升级为Active状态，从而保证集群的正常运行。**ZKFC是HDFS HA自动切换机制的核心**。

## 搭建Hadoop集群(非HA)  
### 准备工作  
1. 服务器说明：  
  
itaojin101: 192.168.1.101 DataNode  
  
itaojin102: 192.168.1.102 DataNode  
  
itaojin103: 192.168.1.103 NamNode  
  
itaojin106: 192.168.1.106 SecondaryNamNode  
2. 每台机器安装SSH，且实现免密登录  
3. 每台机器安装JDK1.8  
  
### 搭建步骤  
#### 下载  
下载Hadoop2.9并解压，将/bin和/sbin加入path环境变量  
#### 配置  
在../etc/hadoop路径下配置文件：  
  
**无特殊说明，以下配置在每个节点上都需要配置**  
  
**code-site.xml**  
  

    <configuration>  
    <!--数据存储路径-->  
    <property>  
    <name>hadoop.tmp.dir</name>  
    <value>/hadoop/tmp</value>  
    <description>Abase for other temporary directories.</description>  
    </property>  
    <!--默认的主节点-->  
    <property>  
    <name>fs.defaultFS</name>  
    <value>hdfs://itaojin103:9000</value>  
    </property>  
    <property>  
    <name>io.file.buffer.size</name>  
    <value>4096</value>  
    </property>  
    </configuration> 

 
  
**hdfs-site.xml**  
  

    <configuration>  
    <!--副本数-->  
    <property>  
    <name>dfs.replication</name>  
    <value>2</value>  
    <description>nodes total count</description>  
    </property>  
      
    <!--只需在NameNode上配置，SecondaryNameNode节点服务器-->  
    <property>  
    <name>dfs.namenode.secondary.http-address</name>  
    <value>itaojin106:50090</value>  
    </property>  
    </configuration>

  
  
新建**mapred-site.xml**  
  

    <configuration>  
    <property>  
    <name>mapreduce.framework.name</name>  
    <value>yarn</value>  
    <final>true</final>  
    </property>  
    <property>  
    <name>mapreduce.jobtracker.http.address</name>  
    <value>itaojin103:50030</value>  
    </property>  
    <property>  
    <name>mapreduce.jobhistory.address</name>  
    <value>itaojin103:10020</value>  
    </property>  
    <property>  
    <name>mapreduce.jobhistory.webapp.address</name>  
    <value>itaojin103:19888</value>  
    </property>  
    <property>  
    <name>mapred.job.tracker</name>  
    <value>http://itaojin103:9001</value>  
    </property>  
    </configuration>

  
  
新建**masters**  
  

    <!--SecondaryNameNode服务器-->  
    itaojin106  

  
**slaves**  
  

    <!--DataNode节点，一行一个-->  
    itaojin101  
    itaojin102  

  
**yarn-site.xml**  
  

    <configuration>  
    <!-- Site specific YARN configuration properties -->  
    <property>  
    <name>yarn.resourcemanager.hostname</name>  
    <value>itaojin103</value>  
    </property>  
    <property>  
    <name>yarn.nodemanager.aux-services</name>  
    <value>mapreduce_shuffle</value>  
    </property>  
    <property>  
    <name>yarn.resourcemanager.address</name>  
    <value>itaojin103:8032</value>  
    </property>  
    <property>  
    <name>yarn.resourcemanager.scheduler.address</name>  
    <value>itaojin103:8030</value>  
    </property>  
    <property>  
    <name>yarn.resourcemanager.resource-tracker.address</name>  
    <value>itaojin103:8031</value>  
    </property>  
    <property>  
    <name>yarn.resourcemanager.admin.address</name>  
    <value>itaojin103:8033</value>  
    </property>  
    <property>  
    <name>yarn.resourcemanager.webapp.address</name>  
    <value>itaojin103:8088</value>  
    </property>  
    </configuration>  

  
**hadoop-env.sh**  
  

    <!--配置本机JDK路径-->  
    export JAVA_HOME=/usr/local/java/jdk1.8.0_152  

  
#### 启动  
只需在NameNode节点上启动即可  
  

    start-all.sh 启动HDFS和MapReduce  
    stop-all.sh  
    start-dfs.sh 只启动HDFS  
    stop-dfs.sh  

  
#### 验证  
  
·NameNode的IP:50070·，进入HDFS管理页面  
  
## 搭建Hadoop集群(HA)

### 环境概要
**HOST**
| 域名 | IP |
|--|--|
| itaojin101 | 192.168.1.101 |
| itaojin102 | 192.168.1.102 |
| itaojin103 | 192.168.1.103 |
| itaojin105 | 192.168.1.105 |
| itaojin106 | 192.168.1.106 |
| itaojin107 | 192.168.1.107 |

**节点规划**
|  | NameNode | JournalNode | DataNode | Zookeeper | ZKFC |
|--|--|--|--|--|--|
| itaojin101 | √ | √ |  |  | √ |
| itaojin102 | √ | √ |  |  | √ |
| itaojin103 |  | √ | √ |  |  |
| itaojin106 |  |  | √ | √ |  |
| itaojin107 |  |  | √ | √ |  |
| itaojin105 |  |  |  | √ |  |

### 环境准备
1. 6台centos7虚拟机
2. 每台服务器上安装好JDK，且保证JDK的安装路径相同，方便后面copy操作。
3. 实现6台服务器root用的SSH免密登录。采用root用户的原因很简单：省事儿。

### 环境搭建

**一下操作先只在一台单机上操作**

#### 下载并解压
地址：[http://apache.claz.org/hadoop/common/hadoop-2.9.0/hadoop-2.9.0.tar.gz](http://apache.claz.org/hadoop/common/hadoop-2.9.0/hadoop-2.9.0.tar.gz)
目录：**cd /usr/local/share/applications** （自定）
下载：**wget http://apache.claz.org/hadoop/common/hadoop-2.9.0/hadoop-2.9.0.tar.gz**
解压：**tar -zxvf  hadoop-2.9.0.tar.gz**

#### 目录介绍
![enter image description here](https://i.imgur.com/rLQEltm.png)

* bin:  可执行脚本
* sbin:  可执行脚本
* etc: 配置文件
* **将 ../share/doc删除，这是说明文档，删除后方便后面服务器之间拷贝**

#### 环境变量
推荐将 **bin** 和 **sbin** 目录添加到path下

#### 配置文件
不推荐直接使用xshell等客户端通过vim修改配置文件，首先很麻烦，再其次易出错。推荐使用EditPlus或者Nodepad++等文本编辑器连接服务器的FTP服务，然后在文本编辑器里进行修改，很方便。

##### hadoop-env.sh
export JAVA_HOME=/usr/local/java/jdk1.8.0_152

##### core-site.xml

    <configuration>
    	<!--指定HDFS的命名空间-->
    	<property>
    	  <name>fs.defaultFS</name>
    	  <value>hdfs://mycluster</value>
    	</property>
    
    	<!--指定HDFS数据存储路径，默认在/tmp下，机器重启后自动删除该目录下的所有数据-->
    	<property>
    		<name>hadoop.tmp.dir</name>
    		<value>/hadoop/tmp</value>
    	</property>
    
    	<!--自动切换备用NameNode所依赖的Zookeeper集群-->
    	<property>
    	   <name>ha.zookeeper.quorum</name>
    	   <value>itaojin105:2181,itaojin106:2181,itaojin107:2181</value>
    	 </property>
    </configuration>

##### hdfs-site.xml

    <configuration>
    	<!--副本数量，默认3-->
    	<property>
    		<name>dfs.replication</name>
    		<value>3</value>
    	</property>
    
    	<!--块大小，默认134217728 byte-->
    	<property>
    		<name>dfs.blocksize</name>
    		<value>134217728</value>
    	</property>
    
    	<!--NameNode的源数据存储路径，默认路径是“file://${hadoop.tmp.dir}/dfs/name”-->
    	<property>
    		<name>dfs.namenode.name.dir</name>
    		<value>/hadoop/tmp/dfs/name</value>
    	</property>
    	
    	<!--DataNode存储数据的路径，默认路径是“file://${hadoop.tmp.dir}/dfs/data”-->
    	<property>
    		<name>dfs.datanode.data.dir</name>
    		<value>/hadoop/tmp/dfs/data</value>
    	</property>
    
    	<!--自定义虚拟服务名称，需要与core-site.xml中的对应上-->
    	<property>
    		<name>dfs.nameservices</name>
    		<value>mycluster</value>
    	</property>
    
    	<!--指定虚拟服务下的NameNode名字，随便取-->
    	<property>
    	  <name>dfs.ha.namenodes.mycluster</name>
    	  <value>nn1,nn2</value>
    	</property>
    
    	<!--NameNode内部节点间的RPC通讯地址-->
    	<property>
    	  <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    	  <value>itaojin101:9000</value>
    	</property>
    	<property>
    	  <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    	  <value>itaojin102:9000</value>
    	</property>
    
    	<!--NameNode对外开放的Http地址-->
    	<property>
    	  <name>dfs.namenode.http-address.mycluster.nn1</name>
    	  <value>itaojin101:50070</value>
    	</property>
    	<property>
    	  <name>dfs.namenode.http-address.mycluster.nn2</name>
    	  <value>itaojin102:50070</value>
    	</property>
    
    	<!--指定JournalNode节点的数据存放目录-->
    	<property>
    	  <name>dfs.namenode.shared.edits.dir</name>
    	  <value>qjournal://itaojin101:8485;itaojin102:8485;itaojin103:8485/mycluster</value>
    	</property>
    
    	<!--指定JournalNode存储本地状态的文件路径-->
    	<property>
    	  <name>dfs.journalnode.edits.dir</name>
    	  <value>/hadoop/tmp/journal/node/local/data</value>
    	</property>
    
    	<!--当NameNode失败时，执行自动切换的主类，能自动将宕掉的NameNode撤下，将standby状态的NameNode切换成Active-->
    	<property>
    	  <name>dfs.client.failover.proxy.provider.mycluster</name>
    	  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    	</property>
    
    	<!--当“脑裂”时，采用“sshfence”方式，且免密地杀死其中一个。什么是“脑裂”：就是两个NameNode都处于Active状态，此种情况Hadoop是不被允许的-->
    	<property>
          <name>dfs.ha.fencing.methods</name>
          <value>sshfence</value>
        </property>
        <property>
          <name>dfs.ha.fencing.ssh.private-key-files</name>
          <value>/root/.ssh/id_rsa</value>
        </property>
    
    	<!--使用SSH免密处理脑裂时的超时时间-->
    	<property>
          <name>dfs.ha.fencing.ssh.connect-timeout</name>
          <value>30000</value>
        </property>
    
    	<!--当NameNode宕掉后，启用自动切换备用NameNode-->
    	<property>
    	   <name>dfs.ha.automatic-failover.enabled</name>
    	   <value>true</value>
    	 </property>
    </configuration>

##### slaves

    itaojin103
    itaojin106
    itaojin107

#### 拷贝
将hadoop整个目录远程拷贝到其他服务器上
如：我在itaojin101上做的以上修改，想将文件夹拷贝到itaojin102上，在itaojin101上执行：

    scp -r /usr/local/share/applications/hadoop-2.9.0 root@itaojin102:/usr/local/share/applications/

#### 初始化

##### 启动Zookeeper
如果Zookeeper集群已经启动则忽略

    cd <zookeeper_home>/bin
    zkServer.sh start

以上操作需在zookeeper集群中的每个节点上执行，运用jps验证是否启动成功。

##### 启动JournalNode节点
通过配置文件可以看出，在itaojin101, itaojin102, itaojin103上部署JournalNode。有两种方式启动JournalNode：
* hadoop-daemons.sh start journalnode  将注册的全部journalnode启动，**但是遇到一个坑--没在itaojin101，itaojin102，itaojin103上启动，而是在itaojin103，itaojin106，itaojin107上，对，跟slaves的配置保持了一致，但是slaves是配置DataNode节点的，目前还不知道为什么，推荐使用第二种方式启动。**
* 在itaojin101，itaojin102，itaojin103上分别执行：

    hadoop-daemons.sh start journalnode

使用jps验证是否启动成功

##### 格式化HDFS
* 在两个NameNode中任意挑选一台来格式化，执行：

    hdfs namenode -format
* 启动NameNode

    hadoop-daemon.sh start namenode

* 在另一台NameNode上去拉取元数据（也可以使用复制）

    hdfs namenode -bootstrapStandby

##### 格式化ZKFC
在任意一台NameNode上执行：

    hdfs zkfc -formatZK

![enter image description here](https://i.imgur.com/iMZtbVX.png)

在zookeeper集群中验证结果：

    <zookeeper_home>/bin/zkCli.sh
    ls /

![enter image description here](https://i.imgur.com/KTc9kbR.png)
**可以看到，zookeeper中多了hadoop-ha**

### 启动

    start-dfs.sh

### 验证
* 用浏览器访问： `http://itaojin101:50070`
* 自动切换
首先访问两个NameNode的50070端口，知道Active和Standby的对应关系，然后到Active的服务器上执行

    **jps | grep NameNode | awk {'print $1'} | xargs kill -9**
    杀死NameNode，此时再访问50070，发现之前的Standby变成了Active，从而验证了自动切换。
    **在自动切换时有一个坑：将Active节点kill掉后，Standby节点不能自动切换，究其原因，原来是最小化的CentOS7没有fuser，需要安装，执行：
yum install psmisc -y
执行以上命令可能报错：“Could not resolve host: centos.ustc.edu.cn; 未知的错误”，原因是没有配置DNS：
    vim /etc/resolv.conf
添加：
    nameserver 8.8.8.8**


## Java客户端访问HDFS
1. 导入jar依赖

> <dependency>  
     <groupId>org.apache.hadoop</groupId>  
     <artifactId>hadoop-client</artifactId>  
     <version>2.9.0</version>  
</dependency>

2. 

    public static void main(String[] args)  {  
        try{  
            Long start = System.currentTimeMillis();  
      Configuration configuration = new Configuration();  
      FileSystem fileSystem = FileSystem.get(new URI("hdfs://itaojin102:9000"), configuration, "root");  
      fileSystem.copyFromLocalFile(new Path("E:\\Maven\\apache-maven-3.5.3"), new Path("/"));  
      fileSystem.close();  
      System.out.println(System.currentTimeMillis()-start + " 毫秒");  
      }catch (Exception ex){  
            ex.printStackTrace();  
      }  
    }

## Shell脚本
[http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html](http://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html)


# Yarn

## Yarn基本服务组件
YARN是Hadoop 2.0中的资源管理[系统](http://www.2cto.com/os/)，它的基本设计思想是将MRv1中的JobTracker拆分成了两个独立的服务：一个全局的资源管理器ResourceManager和每个应用程序特有的ApplicationMaster。其中ResourceManager负责整个[系统](http://www.2cto.com/os/)的资源管理和分配，而ApplicationMaster负责单个应用程序的管理。
![enter image description here](https://images2015.cnblogs.com/blog/669905/201704/669905-20170420115227884-1056505556.png)
YARN总体上仍然是master/slave结构，在整个资源管理框架中，resourcemanager为master，nodemanager是slave。Resourcemanager负责对各个nademanger上资源进行统一管理和调度。当用户提交一个应用程序时，需要提供一个用以跟踪和管理这个程序的ApplicationMaster，它负责向ResourceManager申请资源，并要求NodeManger启动可以占用一定资源的任务。由于不同的ApplicationMaster被分布到不同的节点上，因此它们之间不会相互影响。

YARN的基本组成结构，YARN主要由ResourceManager、NodeManager、ApplicationMaster和Container等几个[组件](http://www.2cto.com/kf/all/zujian/)构成。

ResourceManager是Master上一个独立运行的进程，负责集群统一的资源管理、调度、分配等等；NodeManager是Slave上一个独立运行的进程，负责上报节点的状态；App Master和Container是运行在Slave上的组件，Container是yarn中分配资源的一个单位，包涵内存、CPU等等资源，yarn以Container为单位分配资源。

Client向ResourceManager提交的每一个应用程序都必须有一个Application Master，它经过ResourceManager分配资源后，运行于某一个Slave节点的Container中，具体做事情的Task，同样也运行与某一个Slave节点的Container中。RM，NM，AM乃至普通的Container之间的通信，都是用RPC机制。

YARN的架构设计使其越来越像是一个云操作系统，数据处理操作系统。
![enter image description here](https://images2015.cnblogs.com/blog/669905/201704/669905-20170420115229618-1888016161.jpg)

### Resourcemanager
RM是一个全局的资源管理器，集群只有一个，负责整个系统的资源管理和分配，包括处理客户端请求、启动/监控APP master、监控nodemanager、资源的分配与调度。它主要由两个[组件](http://www.2cto.com/kf/all/zujian/)构成：调度器（Scheduler）和应用程序管理器（Applications Manager，ASM）。

（1） **调度器**

调度器根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。需要注意的是，该调度器是一个“纯调度器”，它不再从事任何与具体应用程序相关的工作，比如不负责监控或者跟踪应用的执行状态等，也不负责重新启动因应用执行失败或者硬件故障而产生的失败任务，这些均交由应用程序相关的ApplicationMaster完成。调度器仅根据各个应用程序的资源需求进行资源分配，而资源分配单位用一个抽象概念“资源容器”（Resource Container，简称Container）表示，Container是一个动态资源分配单位，它将内存、CPU、磁盘、网络等资源封装在一起，从而限定每个任务使用的资源量。此外，该调度器是一个可插拔的组件，用户可根据自己的需要设计新的调度器，YARN提供了多种直接可用的调度器，比如Fair Scheduler和Capacity Scheduler等。

（2） **应用程序管理器**

应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动它等。

### ApplicationMaster（AM）
管理YARN内运行的应用程序的每个实例。

功能：数据切分

为应用程序申请资源并进一步分配给内部任务。

任务监控与容错

负责协调来自resourcemanager的资源，并通过nodemanager监视容易的执行和资源使用情况。

### NodeManager（NM）
Nodemanager整个集群有多个，负责每个节点上的资源和使用。

功能：单个节点上的资源管理和任务。

处理来自于resourcemanager的命令。

处理来自域app master的命令。

Nodemanager管理着抽象容器，这些抽象容器代表着一些特定程序使用针对每个节点的资源。

Nodemanager定时地向RM汇报本节点上的资源使用情况和各个Container的运行状态（cpu和内存等资源）

### Container
Container是YARN中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等，当AM向RM申请资源时，RM为AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能使用该Container中描述的资源。需要注意的是，Container不同于MRv1中的slot，它是一个动态资源划分单位，是根据应用程序的需求动态生成的。目前为止，YARN仅支持CPU和内存两种资源，且使用了轻量级资源隔离机制Cgroups进行资源隔离。

功能：对task环境的抽象

描述一系列信息

任务运行资源的集合（cpu、内存、io等）

任务运行环境

## YARN的资源管理
1. 资源调度和隔离是yarn作为一个资源管理系统，最重要且最基础的两个功能。资源调度由resourcemanager完成，而资源隔离由各个nodemanager实现。

2. Resourcemanager将某个nodemanager上资源分配给任务（这就是所谓的“资源调度”）后，nodemanager需按照要求为任务提供相应的资源，甚至保证这些资源应具有独占性，为任务运行提供基础和保证，这就是所谓的资源隔离。

3. 当谈及到资源时，我们通常指内存、cpu、io三种资源。Hadoop yarn目前为止仅支持cpu和内存两种资源管理和调度。

4. 内存资源多少决定任务的生死，如果内存不够，任务可能运行失败；相比之下，cpu资源则不同，它只会决定任务的快慢，不会对任务的生死产生影响。

### Yarn的内存管理

yarn允许用户配置每个节点上可用的物理内存资源，注意，这里是“可用的”，因为一个节点上内存会被若干个服务贡享，比如一部分给了yarn，一部分给了hdfs，一部分给了hbase等，yarn配置的只是自己可用的，配置参数如下：

    yarn.nodemanager.resource.memory-mb

表示该节点上yarn可以使用的物理内存总量，默认是8192m，注意，如果你的节点内存资源不够8g，则需要调减这个值，yarn不会智能的探测节点物理内存总量。

    yarn.nodemanager.vmem-pmem-ratio

任务使用1m物理内存最多可以使用虚拟内存量，默认是2.1

    yarn.nodemanager.pmem-check-enabled

是否启用一个线程检查每个任务证使用的物理内存量，如果任务超出了分配值，则直接将其kill，默认是true。

    yarn.nodemanager.vmem-check-enabled

是否启用一个线程检查每个任务证使用的虚拟内存量，如果任务超出了分配值，则直接将其kill，默认是true。

    yarn.scheduler.minimum-allocation-mb

单个任务可以使用最小物理内存量，默认1024m，如果一个任务申请物理内存量少于该值，则该对应值改为这个数。

    yarn.scheduler.maximum-allocation-mb

单个任务可以申请的最多的内存量，默认8192m

### Yarn cpu管理

目前cpu被划分为虚拟cpu，这里的虚拟cpu是yarn自己引入的概念，初衷是考虑到不同节点cpu性能可能不同，每个cpu具有计算能力也是不一样的，比如，某个物理cpu计算能力可能是另外一个物理cpu的2倍，这时候，你可以通过为第一个物理cpu多配置几个虚拟cpu弥补这种差异。用户提交作业时，可以指定每个任务需要的虚拟cpu个数。在yarn中，cpu相关配置参数如下：

    yarn.nodemanager.resource.cpu-vcores

表示该节点上yarn可使用的虚拟cpu个数，默认是8个，注意，目前推荐将该值为与物理cpu核数相同。如果你的节点cpu合数不够8个，则需要调减小这个值，而yarn不会智能的探测节点物理cpu总数。

    yarn.scheduler.minimum-allocation-vcores

单个任务可申请最小cpu个数，默认1，如果一个任务申请的cpu个数少于该数，则该对应值被修改为这个数

    yarn.scheduler.maximum-allocation-vcores

单个任务可以申请最多虚拟cpu个数，默认是32.

## 搭建集群 (HA)

### 节点规划

|  | ResourceManager | NodeManager |
|--|--|--|
| itaojin101 | √ |  |
| itaojin102 | √ |  |
| itaojin103 |  | √ |
| itaojin106 |  | √ |
| itaojin107 |  | √ |
| itaojin105 |  |  | 

### 编辑yarn-env.sh

    export JAVA_HOME=/usr/local/java/jdk1.8.0_152

### 编辑yarn-site.xml

    <configuration>
    	<!--是否启用yarn的HA-->
    	<property>
    	  <name>yarn.resourcemanager.ha.enabled</name>
    	  <value>true</value>
    	</property>
    	<!--yarn的HA虚拟服务名-->
    	<property>
    	  <name>yarn.resourcemanager.cluster-id</name>
    	  <value>cluster1</value>
    	</property>
    	<!--yarn的HA虚拟服务名下的具体服务名-->
    	<property>
    	  <name>yarn.resourcemanager.ha.rm-ids</name>
    	  <value>rm1,rm2</value>
    	</property>
    	<!--指定ResourceManager的主机名-->
    	<property>
    	  <name>yarn.resourcemanager.hostname.rm1</name>
    	  <value>itaojin101</value>
    	</property>
    	<property>
    	  <name>yarn.resourcemanager.hostname.rm2</name>
    	  <value>itaojin102</value>
    	</property>
    	<!--指定shuffle-->
    	<property>
    		<name>yarn.nodemanager.aux-services</name>
    		<value>mapreduce_shuffle</value>
    	</property>
    	<!--指定ResourceManager的web UI通讯地址-->
    	<property>
    	  <name>yarn.resourcemanager.webapp.address.rm1</name>
    	  <value>itaojin101:8088</value>
    	</property>
    	<property>
    	  <name>yarn.resourcemanager.webapp.address.rm2</name>
    	  <value>itaojin102:8088</value>
    	</property>
    	<!--指定Zookeeper集群环境-->
    	<property>
    	  <name>yarn.resourcemanager.zk-address</name>
    	  <value>itaojin105:2181,itaojin106:2181,itaojin107:2181</value>
    	</property>
    </configuration>

**至此。Yarn相关的配置已经完成，等到MapReduce配置完毕后一并验证**


# MapReduce

## MapReduce是什么
Hadoop MapReduce是一个软件框架，基于该框架能够容易地编写应用程序，这些应用程序能够运行在由上千个商用机器组成的大集群上，并以一种可靠的，具有容错能力的方式并行地处理上TB级别的海量数据集。这个定义里面有着这些关键词，

* 一是软件框架
* 二是并行处理
* 三是可靠且容错
* 四是大规模集群
* 五是海量数据集

## MapReduce做什么
MapReduce擅长处理大数据，它为什么具有这种能力呢？这可由MapReduce的设计思想发觉。MapReduce的思想就是“**分而治之**”。

　　（1）**Mapper负责“分”**，即把复杂的任务分解为若干个“简单的任务”来处理。“简单的任务”包含三层含义：

一是数据或计算的规模相对原任务要大大**缩小**；二是**就近计算原则**，即任务会分配到存放着所需数据的节点上进行计算；三是这些小任务**可以并行计算，彼此间几乎没有依赖**关系。

　　（2）**Reducer**负责对map阶段的**结果进行汇总**。至于需要多少个Reducer，用户可以根据具体问题，通过在mapred-site.xml配置文件里设置参数mapred.reduce.tasks的值，缺省值为1。

> 一个比较形象的语言解释MapReduce：　　
> 
> 我们要数图书馆中的所有书。你数1号书架，我数2号书架。这就是“**Map**”。我们人越多，数书就更快。
> 
> 现在我们到一起，把所有人的统计数加在一起。这就是“**Reduce**”。

## MapReduce工作机制
![enter image description here](http://images.cnitblog.com/blog/381412/201502/121253045737189.jpg)

MapReduce的整个工作过程如上图所示，它包含如下4个独立的实体：

　　* 实体一：**客户端**，用来提交MapReduce作业。

　　* 实体二：**JobTracker**，用来协调作业的运行。

　　* 实体三：**TaskTracker**，用来处理作业划分后的任务。

　　* 实体四：**HDFS**，用来在其它实体间共享作业文件。

![enter image description here](http://images.cnitblog.com/blog/381412/201502/121311236836173.png)

## Hadoop中的MapReduce框架
一个MapReduce作业通常会把输入的数据集切分为若干独立的数据块，由Map任务以完全并行的方式去处理它们。

框架会对Map的输出先进行排序，然后把结果输入给Reduce任务。通常作业的输入和输出都会被存储在文件系统中，整个框架负责任务的调度和监控，以及重新执行已经关闭的任务。

通常，MapReduce框架和分布式文件系统是运行在一组相同的节点上，也就是说，计算节点和存储节点通常都是在一起的。这种配置允许框架在那些已经存好数据的节点上高效地调度任务，这可以使得整个集群的网络带宽被非常高效地利用。

### MapReduce框架的组成
![enter image description here](http://images.cnitblog.com/blog/381412/201312/21154930-a8557192283247449ce5a4adabc7585d.png)

* （1）JobTracker
　　JobTracker负责调度构成一个作业的所有任务，这些任务分布在不同的TaskTracker上（由上图的JobTracker可以看到2 assign map 和 3 assign reduce）。你可以将其理解为公司的项目经理，项目经理接受项目需求，并划分具体的任务给下面的开发工程师。

* （2）TaskTracker
　　TaskTracker负责执行由JobTracker指派的任务，这里我们就可以将其理解为开发工程师，完成项目经理安排的开发任务即可。

### MapReduce的输入输出
MapReduce框架运转在**<key,value>**键值对上，也就是说，框架把作业的输入看成是一组<key,value>键值对，同样也产生一组<key,value>键值对作为作业的输出，这两组键值对有可能是不同的。

　　一个MapReduce作业的输入和输出类型如下图所示：可以看出在整个流程中，会有三组<key,value>键值对类型的存在。
![enter image description here](http://images.cnitblog.com/blog/381412/201502/121334513709082.png)

### MapReduce的处理流程
这里以WordCount单词计数为例，介绍map和reduce两个阶段需要进行哪些处理。单词计数主要完成的功能是：统计一系列文本文件中每个单词出现的次数，如图所示：

![enter image description here](http://images.cnitblog.com/blog/381412/201502/121405027292936.jpg)

（1）map任务处理
![enter image description here](http://images.cnitblog.com/blog/381412/201502/121403128869360.png)
（2）reduce任务处理
![enter image description here](http://images.cnitblog.com/blog/381412/201502/121414341362568.png)

## 第一个MapReduce程序：WordCount
WordCount单词计数是最简单也是最能体现MapReduce思想的程序之一，该程序完整的代码可以在Hadoop安装包的src/examples目录下找到。

　　WordCount单词计数主要完成的功能是：**统计一系列文本文件中每个单词出现的次数**；

### 初始化一个words.txt文件并上传HDFS
首先在Linux中通过Vim编辑一个简单的words.txt，其内容很简单如下所示：

    Hello Edison Chou
    Hello Hadoop RPC
    Hello Wncud Chou
    Hello Hadoop MapReduce
    Hello Dick Gu
通过Shell命令将其上传到一个指定目录中，这里指定为：/testdir/input

### 自定义Map函数
在Hadoop 中， map 函数位于内置类org.apache.hadoop.mapreduce.**Mapper**<KEYIN,VALUEIN, KEYOUT, VALUEOUT>中，reduce 函数位于内置类org.apache.hadoop. mapreduce.**Reducer**<KEYIN, VALUEIN, KEYOUT, VALUEOUT>中。

　　我们要做的就是**覆盖map 函数和reduce 函数**，首先我们来覆盖map函数：继承Mapper类并重写map方法

    /** * @author Edison Chou
         * @version 1.0
         * @param KEYIN
         *            →k1 表示每一行的起始位置（偏移量offset）
         * @param VALUEIN
         *            →v1 表示每一行的文本内容
         * @param KEYOUT
         *            →k2 表示每一行中的每个单词
         * @param VALUEOUT
         *            →v2 表示每一行中的每个单词的出现次数，固定值为1 */
        public static class MyMapper extends Mapper<LongWritable, Text, Text, LongWritable> { protected void map(LongWritable key, Text value,
                    Mapper<LongWritable, Text, Text, LongWritable>.Context context) throws java.io.IOException, InterruptedException {
                String[] spilted = value.toString().split(" "); for (String word : spilted) {
                    context.write(new Text(word), new LongWritable(1L));
                }
            };
        }

Mapper 类，有四个泛型，分别是KEYIN、VALUEIN、KEYOUT、VALUEOUT，前面两个KEYIN、VALUEIN 指的是map 函数输入的参数key、value 的类型；后面两个KEYOUT、VALUEOUT 指的是map 函数输出的key、value 的类型；

从代码中可以看出，在Mapper类和Reducer类中都使用了Hadoop自带的基本数据类型，例如String对应Text，long对应LongWritable，int对应IntWritable。这是因为HDFS涉及到序列化的问题，Hadoop的基本数据类型都实现了一个Writable接口，而实现了这个接口的类型都支持序列化。

　　这里的map函数中通过空格符号来分割文本内容，并对其进行记录；

### 自定义Reduce函数

现在我们来覆盖reduce函数：继承Reducer类并重写reduce方法

    /** * @author Edison Chou
         * @version 1.0
         * @param KEYIN
         *            →k2 表示每一行中的每个单词
         * @param VALUEIN
         *            →v2 表示每一行中的每个单词的出现次数，固定值为1
         * @param KEYOUT
         *            →k3 表示每一行中的每个单词
         * @param VALUEOUT
         *            →v3 表示每一行中的每个单词的出现次数之和 */
        public static class MyReducer extends Reducer<Text, LongWritable, Text, LongWritable> { protected void reduce(Text key,
                    java.lang.Iterable<LongWritable> values,
                    Reducer<Text, LongWritable, Text, LongWritable>.Context context) throws java.io.IOException, InterruptedException { long count = 0L; for (LongWritable value : values) {
                    count += value.get();
                }
                context.write(key, new LongWritable(count));
            };
        }


Reducer 类，也有四个泛型，同理，分别指的是reduce 函数输入的key、value类型（这里输入的key、value类型通常和map的输出key、value类型保持一致）和输出的key、value 类型。

　　这里的reduce函数主要是将传入的<k2,v2>进行最后的合并统计，形成最后的统计结果。

### 设置Main函数

（1）设定输入目录，当然也可以作为参数传入

public static final String INPUT_PATH = "hdfs://hadoop-master:9000/testdir/input/words.txt";

　　（2）设定输出目录（**输出目录需要是空目录**），当然也可以作为参数传入

public static final String OUTPUT_PATH = "hdfs://hadoop-master:9000/testdir/output/wordcount";

　　（3）Main函数的主要代码
　　

    public static void main(String[] args) throws Exception {
            Configuration conf = new Configuration(); // 0.0:首先删除输出路径的已有生成文件
            FileSystem fs = FileSystem.get(new URI(INPUT_PATH), conf);
            Path outPath = new Path(OUTPUT_PATH); if (fs.exists(outPath)) {
                fs.delete(outPath, true);
            }
    
            Job job = new Job(conf, "WordCount");
            job.setJarByClass(MyWordCountJob.class); // 1.0:指定输入目录
            FileInputFormat.setInputPaths(job, new Path(INPUT_PATH)); // 1.1:指定对输入数据进行格式化处理的类（可以省略）
            job.setInputFormatClass(TextInputFormat.class); // 1.2:指定自定义的Mapper类
            job.setMapperClass(MyMapper.class); // 1.3:指定map输出的<K,V>类型（如果<k3,v3>的类型与<k2,v2>的类型一致则可以省略）
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(LongWritable.class); // 1.4:分区（可以省略）
            job.setPartitionerClass(HashPartitioner.class); // 1.5:设置要运行的Reducer的数量（可以省略）
            job.setNumReduceTasks(1); // 1.6:指定自定义的Reducer类
            job.setReducerClass(MyReducer.class); // 1.7:指定reduce输出的<K,V>类型
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(LongWritable.class); // 1.8:指定输出目录
            FileOutputFormat.setOutputPath(job, new Path(OUTPUT_PATH)); // 1.9:指定对输出数据进行格式化处理的类（可以省略）
            job.setOutputFormatClass(TextOutputFormat.class); // 2.0:提交作业
            boolean success = job.waitForCompletion(true); if (success) {
                System.out.println("Success");
                System.exit(0);
            } else {
                System.out.println("Failed");
                System.exit(1);
            }
        }

在Main函数中，主要做了三件事：一是指定输入、输出目录；二是指定自定义的Mapper类和Reducer类；三是提交作业；匆匆看下来，代码有点多，但有些其实是可以省略的。

　　（4）完整代码如下所示

### 运行吧小DEMO
（1）调试查看控制台状态信息
![enter image description here](http://images.cnitblog.com/blog/381412/201502/121454532613812.jpg)

（2）通过Shell命令查看统计结果
![enter image description here](http://images.cnitblog.com/blog/381412/201502/121456144951094.jpg)

## 使用ToolRunner类改写WordCount
Hadoop有个ToolRunner类，它是个好东西，简单好用。无论在《Hadoop权威指南》还是Hadoop项目源码自带的example，都推荐使用ToolRunner。

### 最初的写法
下面我们看下src/example目录下WordCount.java文件，它的代码结构是这样的：

    public class WordCount { // 略...
        public static void main(String[] args) throws Exception {
            Configuration conf = new Configuration();
            String[] otherArgs = new GenericOptionsParser(conf, 
                                                args).getRemainingArgs(); // 略...
            Job job = new Job(conf, "word count"); // 略...
            System.exit(job.waitForCompletion(true) ? 0 : 1);
        }
    }
WordCount.java中使用到了GenericOptionsParser这个类，它的作用是**将命令行中参数自动设置到变量conf中**。举个例子，比如我希望通过命令行设置reduce task数量，就这么写：

bin/hadoop jar MyJob.jar com.xxx.MyJobDriver -Dmapred.reduce.tasks=5

　　上面这样就可以了，不需要将其硬编码到java代码中，很轻松就可以将参数与代码分离开。

### 加入ToolRunner的写法

至此，我们还没有说到ToolRunner，上面的代码我们使用了GenericOptionsParser帮我们解析命令行参数，编写ToolRunner的程序员更懒，它将 GenericOptionsParser调用隐藏到自身run方法，被自动执行了，修改后的代码变成了这样：

    public class WordCount extends Configured implements Tool {
        @Override public int run(String[] arg0) throws Exception {
            Job job = new Job(getConf(), "word count"); // 略...
            System.exit(job.waitForCompletion(true) ? 0 : 1); return 0;
        } public static void main(String[] args) throws Exception { int res = ToolRunner.run(new Configuration(), new WordCount(), args);
            System.exit(res);
        }
    }
看看这段代码上有什么不同：

　　（1）让WordCount**继承Configured并实现Tool接口**。

　　（2）**重写Tool接口的run方法**，run方法不是static类型，这很好。

　　（3）在WordCount中我们将**通过getConf()获取Configuration对象**。

　　可以看出，通过简单的几步，就可以实现代码与配置隔离、上传文件到DistributeCache等功能。修改MapReduce参数不需要修改java代码、打包、部署，提高工作效率。

### 重写WordCount程序

    public class MyJob extends Configured implements Tool { public static class MyMapper extends Mapper<LongWritable, Text, Text, LongWritable> { protected void map(LongWritable key, Text value,
                    Mapper<LongWritable, Text, Text, LongWritable>.Context context) throws java.io.IOException, InterruptedException {
                           ......
                }
            };
        } public static class MyReducer extends Reducer<Text, LongWritable, Text, LongWritable> { protected void reduce(Text key,
                    java.lang.Iterable<LongWritable> values,
                    Reducer<Text, LongWritable, Text, LongWritable>.Context context) throws java.io.IOException, InterruptedException {
                           ......
            };
        } // 输入文件路径
        public static final String INPUT_PATH = "hdfs://hadoop-master:9000/testdir/input/words.txt"; // 输出文件路径
        public static final String OUTPUT_PATH = "hdfs://hadoop-master:9000/testdir/output/wordcount";
    
        @Override public int run(String[] args) throws Exception { // 首先删除输出路径的已有生成文件
            FileSystem fs = FileSystem.get(new URI(INPUT_PATH), getConf());
            Path outPath = new Path(OUTPUT_PATH); if (fs.exists(outPath)) {
                fs.delete(outPath, true);
            }
    
            Job job = new Job(getConf(), "WordCount"); // 设置输入目录
            FileInputFormat.setInputPaths(job, new Path(INPUT_PATH)); // 设置自定义Mapper
            job.setMapperClass(MyMapper.class);
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(LongWritable.class); // 设置自定义Reducer
            job.setReducerClass(MyReducer.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(LongWritable.class); // 设置输出目录
            FileOutputFormat.setOutputPath(job, new Path(OUTPUT_PATH));
    
            System.exit(job.waitForCompletion(true) ? 0 : 1); return 0;
        } public static void main(String[] args) {
            Configuration conf = new Configuration(); try { int res = ToolRunner.run(conf, new MyJob(), args);
                System.exit(res);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    
    }


## 配置
### mapred-site.cml

    <configuration>
    	<!--指定mapreduce的运行框架-->
    	<property>
    		<name>mapreduce.framework.name</name>
    		<value>yarn</value>
    		<final>true</final>
    	</property>
    </configuration>

## 启动
1. 在itaojin101上执行：

    start-yarn.sh
2. 此时三个NodeManager节点已经启动，且itaojin101上的ResourceManager节点也已经启动，需要手动到itaojin102终端启动ResourceManager：

    yarn-daemon.sh start resourcemanager

至此，高可用MapReduce集群已经搭建完毕！

## 验证
1. 运行 `jps`查看各个服务器上的相关节点，如下图所示就对了：

|  | NameNode | JournalNode | DataNode | Zookeeper | ZKFC | ResourceManager | NodeManager |
|--|--|--|--|--|--|--|--|
| itaojin101 | √ | √ |  |  | √ | √ |  |
| itaojin102 | √ | √ |  |  | √ | √ |  |
| itaojin103 |  | √ | √ |  |  |  | √ |
| itaojin106 |  |  | √ | √ |  |  | √ |
| itaojin107 |  |  | √ | √ |  |  | √ |
| itaojin105 |  |  |  | √ |  |  |  |

2. 通过运行一个WordCount小程序来验证：

* 在本地/root/下新建一个WordCount文件，并编辑为：

    hadoop is nice
    hadoop good
    hadoop is better
* 将WordCount文件上传至HDFS根目录

     hdfs dfs -put /root/WordCount /
* 执行命令：
	

    yarn jar <hadoop_home>/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.9.0.jar wordcount /WordCount /out
**运用Hadoop自带的wordcount统计器统计hdfs文件根目录下的WordCount文件，并且将结果输出到/out目录下**
* 查看结果
![enter image description here](https://i.imgur.com/U8jTQD5.png)

3. 验证高可用
**将itaojin101上的ResourceManager进程杀死，itaojin102上的ResourceManager从Standby状态变成Active**
通过以下命令查看节点身份：

    yarn rmadmin -getServiceState rm1
