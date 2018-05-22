# 搭建Hadoop集群(非HA)
## 准备工作
1. 服务器说明：

    itaojin101: 192.168.1.101 DataNode

    itaojin102: 192.168.1.102 DataNode

    itaojin103: 192.168.1.103 NamNode

    itaojin106: 192.168.1.106 SecondaryNamNode
2. 每台机器安装SSH，且实现免密登录
3. 每台机器安装JDK1.8

## 搭建步骤
### 下载
下载Hadoop2.9并解压，将/bin和/sbin加入path环境变量
### 配置
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

### 启动
只需在NameNode节点上启动即可

    start-all.sh 启动HDFS和MapReduce
    stop-all.sh 
    start-dfs.sh 只启动HDFS
    stop-dfs.sh

### 验证

·NameNode的IP:50070·，进入HDFS管理页面