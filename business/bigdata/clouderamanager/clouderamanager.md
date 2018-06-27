# 前言
以下记录是笔者在使用ClouderaManager的过程中的一些经验心得和踩过的坑，希望能对部门的小伙伴们有所帮助。由于笔者刚开始接触大数据，水平有限，肯定会有不当之处，望各位小伙伴指正。联系方式：[fuqq@itaojin.cn](fuqq@itaojin.cn)

# Hive
使用ClouderaManager安装Hive。
这里的ClouderaManager版本是5.14，支持的Hive只有2.x系列。
## 环境准备
默认至少已经具备以下条件：
* HDFS集群，最好是HA（高可用）
* YARN集群，最好是HA
* Zookeeper集群

## 安装
* 添加新服务
![enter image description here](https://i.imgur.com/YMsmFRX.png)

* 选择Hive
![enter image description here](https://i.imgur.com/FCXorla.png)

* 选择一组依赖
![enter image description here](https://i.imgur.com/ypGHbsR.png)

* 为Hive生态中的每种服务分配节点资源。这里列举了 `Gateway` , `Metastore Server` , `WebHCat Server` , `HiveServer2` 四种服务。
	1. Gateway。目前对其了解的不多，其用途大概是提供修改Hive配置文件的客户端，在选择节点时，笔者是在HiveServer2节点上都创建了该节点，理由是对Hive的配置修改其实就是对HiveServer2服务的配置的修改。
	2. Metastore Service。元数据服务，即Hive存放所有表格，试图，函数等等元数据的服务。由于该服务存储了大量元数据，故而对内存和稳定性要求较高，理论上该服务可以放置于任何节点，最好该节点配置要稍好。
	3. WebHCat Server。未知。
	4. HiveServer2。Hive提供服务的服务。支持JDBC，shell等方式访问服务。对该服务的探索还在继续中，该服务不需要在每个节点上部署，视业务需求而定。		
	![enter image description here](https://i.imgur.com/rneuDVI.png)

* 为Metastore Server节点配置Mysql数据库。需要事先在某一台节点上的Mysql中创建数据库和用户，推荐跟`scm`在同一个Mysql。
	

	> create database hive; 	

	> create user 'hive' identified by 'hive'; 
		
	> grant all privileges on hive.* to 'hive'@'localhost' identified by 'hive';

	> 	grant all privileges on hive.* to 'hive'@'%' identified by 'hive';

	> 	flush privileges;

	**在这里需要注意，如果是首次安装，还需要完成重要的一步：将Mysql驱动jar添加到`/usr/share/java`目录下，且改名为`mysql-connector-java.jar`，即去掉版本号。这是至关重要的一步，不然Hive将不能连接Mysql，也可以推出，Hive是通过JDBC的方式访问Mysql。**
	
	驱动下载地址：[下载地址](http://192.168.1.105:8081/nexus/content/groups/public/mysql/mysql-connector-java/5.1.38/mysql-connector-java-5.1.38.jar)

	添加完成后，点击`测试连接`按钮完成连接测试。
	
	![enter image description here](https://i.imgur.com/FssVckP.png)

* 为Hive指定在HDFS中的数据目录，保持默认状态，下一步
![enter image description here](https://i.imgur.com/NVZkItG.png)

* 开始安装
	![enter image description here](https://i.imgur.com/pE7kmSV.png)

* 完成
	![enter image description here](https://i.imgur.com/di7ar5B.png)

* 回到CM首页即可看到安装完毕的Hive服务
	![enter image description here](https://i.imgur.com/wWcJR6I.png)

## 测试
在安装了`HiveServer2`服务的节点上执行以下shell脚本：

> beeline

调出Hive客户端，连接HiveServer2服务：

> beeline> !connect jdbc:hive2://localhost:10000

回车两次即可
![enter image description here](https://i.imgur.com/7l3yiL3.png)

展示所有database

> show databases;

展现出默认`default`数据库，测试完毕！

## 注意
### MySQL驱动jar
这个问题已经在前面讲过，不再赘述。

### MySQL中user表中的默认记录导致的一个错误
* 业务场景：ClouderaManager安装Hive时需要一个MySQL数据库存放元数据，于是创建hive库和hive用户，并且授权，操作命令如下：
		
	> create database hive; 	
		
	> create user 'hive'  identified by 'hive';

	> grant all privileges on hive.* to 'hive'@'localhost' identified by 'hive'; 		

	> grant all privileges on hive.* to 'hive'@'%' identified by 'hive'; 		

	> flush privileges;

	通过远程连接hive库ok，但是在ClouderaManager上安装Hive时就悲剧了，直接报错：

	  Logon denied for user/password. Able to find the database server and database, but the login reques was rejected

	![enter image description here](https://i.imgur.com/s5VkTDV.png)

* 解决：以root用户登录MySQL，查询user表得知：
	![enter image description here](https://i.imgur.com/ZHTLzth.png)

	可以看到有两行记录的User列为空，没错，就是因为这个报的错，直接将这两行记录删除即可：
	
	> delete from user where User = ''; 	

	> flush privileges;

	**别忘了执行最后一句SQL刷新权限哦**
	

### 权限问题
ClouderaManager安装HDFS后会在操作系统里创建一个hdfs用户，且HDFS文件系统中的根目录拥有者也是hdfs，例如根目录下的`/user`目录权限是用户`hdfs`所有，hive客户端在HDFS中对应的用户是`anonymous`，如果使用hive客户端插入数据，会报权限不足的问题：

    0: jdbc:hive2://localhost:10000> insert into student values (1, 'zhangsan', 23);
    INFO  : Compiling command(queryId=hive_20180620114040_16eb72bd-3d66-46a5-802a-695a5d4963b3): insert into student values (1, 'zhangsan', 23)
    INFO  : Semantic Analysis Completed
    INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:_col0, type:int, comment:null), FieldSchema(name:_col1, type:string, comment:null), FieldSchema(name:_col2, type:int, comment:null)], properties:null)
    INFO  : Completed compiling command(queryId=hive_20180620114040_16eb72bd-3d66-46a5-802a-695a5d4963b3); Time taken: 1.142 seconds
    INFO  : Executing command(queryId=hive_20180620114040_16eb72bd-3d66-46a5-802a-695a5d4963b3): insert into student values (1, 'zhangsan', 23)
    INFO  : Query ID = hive_20180620114040_16eb72bd-3d66-46a5-802a-695a5d4963b3
    INFO  : Total jobs = 3
    INFO  : Launching Job 1 out of 3
    INFO  : Starting task [Stage-1:MAPRED] in serial mode
    INFO  : Number of reduce tasks is set to 0 since there's no reduce operator
    ERROR : Job Submission failed with exception 'org.apache.hadoop.security.AccessControlException(Permission denied: user=anonymous, access=WRITE, inode="/user":hdfs:supergroup:drwxr-xr-x
    	at org.apache.hadoop.hdfs.server.namenode.DefaultAuthorizationProvider.checkFsPermission(DefaultAuthorizationProvider.java:279)
    	at org.apache.hadoop.hdfs.server.namenode.DefaultAuthorizationProvider.check(DefaultAuthorizationProvider.java:260)
    	at org.apache.hadoop.hdfs.server.namenode.DefaultAuthorizationProvider.check(DefaultAuthorizationProvider.java:240)
    	at org.apache.hadoop.hdfs.server.namenode.DefaultAuthorizationProvider.checkPermission(DefaultAuthorizationProvider.java:162)
    	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:152)
    	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:3877)
    	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:3860)
    	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkAncestorAccess(FSDirectory.java:3842)
    	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkAncestorAccess(FSNamesystem.java:6817)
    	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.mkdirsInternal(FSNamesystem.java:4559)
    	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.mkdirsInt(FSNamesystem.java:4529)
    	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.mkdirs(FSNamesystem.java:4502)
    	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.mkdirs(NameNodeRpcServer.java:884)
    	at org.apache.hadoop.hdfs.server.namenode.AuthorizationProviderProxyClientProtocol.mkdirs(AuthorizationProviderProxyClientProtocol.java:328)
    	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.mkdirs(ClientNamenodeProtocolServerSideTranslatorPB.java:641)
    	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
    	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:617)
    	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:1073)
    	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2281)
    	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2277)
    	at java.security.AccessController.doPrivileged(Native Method)
    	at javax.security.auth.Subject.doAs(Subject.java:422)
    	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1920)
    	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2275)

目前还不知道如何根治这个问题，但有一个解决方案：

切换`hdfs`用户

> su hdfs

授权

> hdfs dfs -chmod -R 777 /user

这样做不好的地方就是改变了HDFS文件目录的权限，目前还不知道有没有后遗症，但是解决权限问题。

### 安装Hive的一个坑
当ClouderaManager第一次安装Hive是好使的，如果将Hive删除后重新安装，会出现一些问题：**Hive会使用内置的derby数据库，即便在创建Hive时指定了外部MySQL数据库**，直接影响到注入sqoop从关系型数据导数据到Hive的成功性，这究竟是为什么呢？

还得先从ClouderaManager环境下Hive的安装目录说起。Hive的安装目录在外部提供的parcels包里，具体位置是：`/opt/cloudera/parcels/CDH-5.15.0-1.cdh5.15.0.p0.21/lib/hive`，这里将外部parcels放到了 `/opt/cloudea`文件夹下，根据自己环境的实际安装路径而定。
![enter image description here](https://i.imgur.com/AHvXf1o.png)

从图中可以看到，hive安装目录下的`conf`是引用了 `/etc/hive/conf` 的软连接（*软连接相关知识百度下，大概意思就是创建一个对某个文件夹的引用，并指向另一个虚拟文件夹，物理文件夹的变动实时反映到虚拟文件夹*），我们直接到该目录下一探究竟。
![enter image description here](https://i.imgur.com/c7VYDaa.png)
原来`/etc/hive/conf`又是同目录下`conf.cloudera.hive`目录的软连接。其实，刚开始的模样并非如此，这也是这个坑的出处。

当安装好ClouderaManager后，在 `/etc/hive` 只存在一个软连接文件夹 `conf`，并且指向 `/etc/alternatives/hive-conf`，

![enter image description here](https://i.imgur.com/iu6H99n.png)
`/etc/alternatives/hive-conf`目录下的内容是：
![enter image description here](https://i.imgur.com/22gbGm5.png)
查看 `hive-site.xml` 文件：

    <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:derby:;databaseName=/var/lib/hive/metastore/metastore_db;create=true</value>
      <description>JDBC connect string for a JDBC metastore</description>
    </property>
    
    <property>
      <name>javax.jdo.option.ConnectionDriverName</name>
      <value>org.apache.derby.jdbc.EmbeddedDriver</value>
      <description>Driver class name for a JDBC metastore</description>
    </property>

该配置文件只定义了Hive的内置derby数据库，这也是Hive在ClouderaManager环境下的初始配置。

当第一次安装并启动Hive时，`/etc/hive/`目录下会产生一个文件夹 `conf.cloudera.hive`，该文件夹推测是ClouderaManager结合当前环境整合的一个Hive配置集合，进入目录：
![enter image description here](https://i.imgur.com/Y3XQ2QD.png)
可以看到，新产生的目录下存放着hadoop的一些配置文件和一个`hive-site.xml`文件，查看`hive-site.xml`：

    <configuration>
      <property>
        <name>hive.metastore.uris</name>
        <value>thrift://node5:9083</value>
      </property>
      <property>
        <name>hive.metastore.client.socket.timeout</name>
        <value>300</value>
      </property>
      <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
      </property>
      <property>
        <name>hive.warehouse.subdir.inherit.perms</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.auto.convert.join</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.auto.convert.join.noconditionaltask.size</name>
        <value>20971520</value>
      </property>
      <property>
        <name>hive.optimize.bucketmapjoin.sortedmerge</name>
        <value>false</value>
      </property>
      <property>
        <name>hive.smbjoin.cache.rows</name>
        <value>10000</value>
      </property>
      <property>
        <name>hive.server2.logging.operation.enabled</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.server2.logging.operation.log.location</name>
        <value>/var/log/hive/operation_logs</value>
      </property>
      <property>
        <name>mapred.reduce.tasks</name>
        <value>-1</value>
      </property>
      <property>
        <name>hive.exec.reducers.bytes.per.reducer</name>
        <value>67108864</value>
      </property>
      <property>
        <name>hive.exec.copyfile.maxsize</name>
        <value>33554432</value>
      </property>
      <property>
        <name>hive.exec.reducers.max</name>
        <value>1099</value>
      </property>
      <property>
        <name>hive.vectorized.groupby.checkinterval</name>
        <value>4096</value>
      </property>
      <property>
        <name>hive.vectorized.groupby.flush.percent</name>
        <value>0.1</value>
      </property>
      <property>
        <name>hive.compute.query.using.stats</name>
        <value>false</value>
      </property>
      <property>
        <name>hive.vectorized.execution.enabled</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.vectorized.execution.reduce.enabled</name>
        <value>false</value>
      </property>
      <property>
        <name>hive.merge.mapfiles</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.merge.mapredfiles</name>
        <value>false</value>
      </property>
      <property>
        <name>hive.cbo.enable</name>
        <value>false</value>
      </property>
      <property>
        <name>hive.fetch.task.conversion</name>
        <value>minimal</value>
      </property>
      <property>
        <name>hive.fetch.task.conversion.threshold</name>
        <value>268435456</value>
      </property>
      <property>
        <name>hive.limit.pushdown.memory.usage</name>
        <value>0.1</value>
      </property>
      <property>
        <name>hive.merge.sparkfiles</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.merge.smallfiles.avgsize</name>
        <value>16777216</value>
      </property>
      <property>
        <name>hive.merge.size.per.task</name>
        <value>268435456</value>
      </property>
      <property>
        <name>hive.optimize.reducededuplication</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.optimize.reducededuplication.min.reducer</name>
        <value>4</value>
      </property>
      <property>
        <name>hive.map.aggr</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.map.aggr.hash.percentmemory</name>
        <value>0.5</value>
      </property>
      <property>
        <name>hive.optimize.sort.dynamic.partition</name>
        <value>false</value>
      </property>
      <property>
        <name>hive.execution.engine</name>
        <value>mr</value>
      </property>
      <property>
        <name>spark.executor.memory</name>
        <value>3039297536</value>
      </property>
      <property>
        <name>spark.driver.memory</name>
        <value>966367641</value>
      </property>
      <property>
        <name>spark.executor.cores</name>
        <value>4</value>
      </property>
      <property>
        <name>spark.yarn.driver.memoryOverhead</name>
        <value>102</value>
      </property>
      <property>
        <name>spark.yarn.executor.memoryOverhead</name>
        <value>511</value>
      </property>
      <property>
        <name>spark.dynamicAllocation.enabled</name>
        <value>true</value>
      </property>
      <property>
        <name>spark.dynamicAllocation.initialExecutors</name>
        <value>1</value>
      </property>
      <property>
        <name>spark.dynamicAllocation.minExecutors</name>
        <value>1</value>
      </property>
      <property>
        <name>spark.dynamicAllocation.maxExecutors</name>
        <value>2147483647</value>
      </property>
      <property>
        <name>hive.metastore.execute.setugi</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.support.concurrency</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.zookeeper.quorum</name>
        <value>node2,node6,node5</value>
      </property>
      <property>
        <name>hive.zookeeper.client.port</name>
        <value>2181</value>
      </property>
      <property>
        <name>hive.zookeeper.namespace</name>
        <value>hive_zookeeper_namespace_hive</value>
      </property>
      <property>
        <name>hive.cluster.delegation.token.store.class</name>
        <value>org.apache.hadoop.hive.thrift.MemoryTokenStore</value>
      </property>
      <property>
        <name>hive.server2.enable.doAs</name>
        <value>true</value>
      </property>
      <property>
        <name>hive.server2.use.SSL</name>
        <value>false</value>
      </property>
      <property>
        <name>spark.shuffle.service.enabled</name>
        <value>true</value>
      </property>
    </configuration>
ClouderaManager已经为我们生成了一份Hive配置文件，并且 `/etc/hive/conf` 目录的软连接指向已经变成了这个新增的目录 `/etc/hive/conf.cloudera.hive`，显而易见，Hive已经将配置文件路径指向了新生成的目录下。当我们自己搭建源生Hive环境时，以上配置也可以作为有力参考。配置中着重要注意第一个配置`hive.metastore.uris`，此配置是指定元数据服务器路径。

讲了这么多，以上内容只是内容铺垫，下面正式进入主题。

当把已经装好的Hive服务从ClouderaManager平台移除，且重新安装时，问题就来了，`/etc/hive/conf`软连接地址又指回了初始值 `/etc/alternatives/hive-conf`，使得Hive服务从内置数据库derby寻找数据源，从而报找不到数据库的错误，其本质就是找不到元数据服务了，所以解决办法就是手动创建软连接，重新指向同目录下的`conf.cloudera.hive`目录：

> cd /etc/hive/conf;
 
> rm -rf conf;
 
> ln -s conf.cloudera.hive conf;

重启Hive服务即可。

找不到数据库报错会影响以下两方面流程：
* 不能从外部直接导数据到Hive，例如sqoop
* 当Hive和HBase存在关联表时，也不能从外部导数据到HBase


# Sqoop1
ClouderaManager默认已经安装了Sqoop1组件，且已经配置了环境变量，直接使用即可。

Sqoop1的使用不再赘述，相关章节已经有说明，下面主要讲讲在导数据过程中遇到的一些问题。

## 导入HBase
当把数据从关系型数据库导入HBase时，HBase中的表既可以事先建好，也可以让sqoop自动映射新建。

在关系型数据库中，表的主键大致可以分为 **单主键** 和 **联合主键**，并且还要考虑到表 **无主键** 的情况，所以在导入HBase时都要考虑 到。

### 单主键
通过 `--hbase-row-key <row_key>` 来指定HBase表中的rowKey。

### 联合主键
如果关系型数据库中出现联合主键，则可以通过 `--hbase-row-key <row_key_1>, <row_key_2>[...]`来制定联合主键，HBase会把多个主键以 `_` 的形式拼接起来作为rowKey，但是联合主键就不在正常字段中展示，如果想让联合主键在正常字段中冗余存在，可以在Sqoop的`sqoop-site.xml`文件中添加如下配置：

    <property>
            <name>sqoop.hbase.add.row.key</name>
            <value>true</value>
    </property>
ClouderaManager环境下，该文件在 `/etc/sqoop/conf` 目录下。

如果HBase整合了Hive，那么在创建Hive的表时，需要在原有字段的基础上新增一个主键字段，没错，该字段就是对应的HBase中的rowKey，由联合主键的成员以 `_` 拼接而成。