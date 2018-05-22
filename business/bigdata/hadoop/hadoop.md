**Hadoop2.9.0 主要由HDFS，Yarn和MapReduce三部分构成**

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



