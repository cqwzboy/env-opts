# 原理和架构

原文链接：
[https://blog.csdn.net/matthewei6/article/details/50554472](https://blog.csdn.net/matthewei6/article/details/50554472)
[http://www.infoq.com/cn/articles/introductionOozie/](http://www.infoq.com/cn/articles/introductionOozie/)
[https://www.cnblogs.com/juncaoit/p/6122187.html](https://www.cnblogs.com/juncaoit/p/6122187.html)

## Oozie是什么
Oozie众多大数据工作流调度框架中的一种。

大数据工作流调度概念：一组复杂的计算需要多个环节组成，每个环节用到大数据生态中的一个或者多个组件配合完成，工作流调度工作就是将这些纷繁复杂的组件有序的组合并完成任务。

## 同类
* Oozie （已经成为hadoop标配）
Yahoo！开源，基于xml表达作业依赖关系；

* Azkaban
Linkedin开源，通过Java property配置作业依赖关系

* Zeus（宙斯） （据说不再更新）
阿里开源，通过界面配置作业依赖关系

* 其他开源系统
Cascading（通过Java API编程实现作业依赖关系）

## Oozie三大功能
* **Oozie Workflow jobs** ：工作流任务，可以生成DAG图
* **Oozie Coordinator jobs**：可以定时调度
* **Oozie Bundle**：多个coordinator的集合，或者多个workflow的集合

## Oozie节点
节点，即完成每一步工作的具体执行者。
分类：
* 控制流节点：起始，分支，并发，汇合，结束
* 动作节点（action）：执行的job。例如，mapreduce action，hive action ，shell action

![enter image description here](https://i.imgur.com/Yct0J3Y.png)

## 架构
![enter image description here](https://i.imgur.com/bb7UYwx.png)

* 左边：Oozie提供完整的API接口，包括java API，REST API，webUI（只读），Oozie CLI 等等
* 中间：Oozie服务器是一个java的tomcat服务，可以直线**三大功能**
* 右边：直接操作HDFS和MR

## 运行流程
![enter image description here](https://i.imgur.com/od4WA5K.jpg)

## hPDL
Oozie工作流是放置在控制依赖DAG（有向无环图 Direct Acyclic Graph）中的一组动作（例如，Hadoop的Map/Reduce作业、Pig作业等），其中指定了动作执行的顺序。我们会使用hPDL（一种XML流程定义语言）来描述这个图。

hPDL是一种很简洁的语言，只会使用少数流程控制和动作节点。控制节点会定义执行的流程，并包含工作流的起点和终点（start、end和fail节点）以及控制工作流执行路径的机制（decision、fork和join节点）。动作节点是一些机制，通过它们工作流会触发执行计算或者处理任务。Oozie为以下类型的动作提供支持： Hadoop map-reduce、Hadoop文件系统、Pig、Java和Oozie的子工作流（SSH动作已经从Oozie schema 0.2之后的版本中移除了）。

所有由动作节点触发的计算和处理任务都不在Oozie之中——它们是由Hadoop的Map/Reduce框架执行的。这种方法让Oozie可以支持现存的Hadoop用于负载平衡、灾难恢复的机制。这些任务主要是异步执行的（只有文件系统动作例外，它是同步处理的）。这意味着对于大多数工作流动作触发的计算或处理任务的类型来说，在工作流操作转换到工作流的下一个节点之前都需要等待，直到计算或处理任务结束了之后才能够继续。Oozie可以通过两种不同的方式来检测计算或处理任务是否完成，也就是回调和轮询。当Oozie启动了计算或处理任务的时候，它会为任务提供唯一的回调URL，然后任务会在完成的时候发送通知给特定的URL。在任务无法触发回调URL的情况下（可能是因为任何原因，比方说网络闪断），或者当任务的类型无法在完成时触发回调URL的时候，Oozie有一种机制，可以对计算或处理任务进行轮询，从而保证能够完成任务。


# 安装

原文链接：
[https://www.cnblogs.com/duanxingxing/p/5015709.html](https://www.cnblogs.com/duanxingxing/p/5015709.html)
[https://blog.csdn.net/u014729236/article/details/47188631](https://blog.csdn.net/u014729236/article/details/47188631)

对比大数据生态的其他组件的安装过程，Oozie的安装过程是略显复杂的，最突出的一点就是Apache官方只提供Oozie的源码，不提供已经打好的tar包，而且编译过程有很多不确定性因素，这就抬高了门槛。

经验之谈：在安装各个组件时，建议不要安装最新的版本，不管是从兼容性考虑，还是从参考资料的完整性考虑皆是如此，秉承**够用原则**。

## 环境介绍：
* JDK1.8
* Maven  3.5.3
* Hadoop 2.6.0集群
* Oozie 4.3.0

## 下载源码
直接到官网下载：[http://archive.apache.org/dist/oozie/4.3.0/](http://archive.apache.org/dist/oozie/4.3.0/)

> cd /usr/local/share/applications;
> wget http://archive.apache.org/dist/oozie/4.3.0/oozie-4.3.0.tar.gz;
> tar -zxvf oozie-4.3.0.tar.gz;
> mv oozie-4.3.0 oozie-4.3.0-main;

之所以要将oozie-4.3.0重命名，是因为后面会重新编译并解压出一个oozie-4.3.0文件夹，为了避免覆盖。

## 检查Maven配置
如果机器上安装的Maven配置了第三方源或者私服，请屏蔽，因为Oozie的源码里已经指定了maven中央仓库源，且Oozie的很多依赖jar只有中央仓库才有。**如果没有配置第三方源或者私服，跳过此步。**

修改命令：
> vim $MAVEN_HOME/conf/settings.xml
* 将**mirrors**节点下镜像屏蔽
* 将**activeProfiles**开关屏蔽

## 编译

> cd ./oozie-4.3.0-main/bin;
> ./mkdistro.sh -DskipTests;

进入源码根目录下的bin目录，执行编译，**-DskipTests**为跳过编译。
编译的过程很漫长，大概要持续**15~20**分钟。并且编译过程中会报找不到一些jar，**只要编译过程没有中断，可以不理**。

Tips：

> 刚开始Hadoop的版本是很新的2.9.0，在编译oozie的过程中发现很多jar找不到，于是我将Hadoop的版本降为2.6.0才正常了。

## 安装Oozie Server
### 解压
完成**编译**后，会在 `/usr/local/share/applications/oozie-4.3.0-main/distro/target` 下生成一个tar包：**oozie-4.3.0-distro.tar.gz**，没错，这就是编译好的安装包，将其解压到与oozie-4.3.0-main平级的目录下：

> cd /usr/local/share/applications/oozie-4.3.0-main/distro/target tar;
> -zxvf oozie-4.3.0-distro.tar.gz -C /usr/local/share/applications/;

我们在`/usr/local/share/applications/`目录下可以看到有一个**oozie-4.3.0** 的文件夹

### 环境变量
将Oozie的根目录加到/etc/profile文件中

> vim /etc/profile
![enter image description here](https://i.imgur.com/SozCIWJ.png)
> source /etc/profile

### 新建libext
进入**oozie-4.3.0**，新建文件夹**libext**，此文件夹存放Oozie Server依赖的jar，且名称只能是**libext**，不可更改，应该是源码中会依据此名称文件夹进行解析。

> cd /usr/local/share/applications/oozie-4.3.0 ; 
> mkdir libext;

jar依赖一共分为三种类型：

#### ext-2.2.zip
为Oozie的webUI提供JS
下载地址：[https://ext4all.com/post/how-to-download-extjs-2.html](https://ext4all.com/post/how-to-download-extjs-2.html)

> cd libext;
> wget https://ext4all.com/ext/download/ext-2.2.zip ;
> unzip ext-2.2.zip;

如果报不识别**unzip**，先安装unzip，centOS命令如下：

> yum install zip unzip -y;

#### mysql-connector-java-5.1.32.jar
由于Oozie的任务数据存放在数据库中，这里我们采用Mysql数据库存放，所以需要导入驱动jar

    wget http://itaojin105:8081/nexus/content/groups/public/mysql/mysql-connector-java/5.1.32/mysql-connector-java-5.1.32.jar;

这里是从**itaojin105**的Maven私服上获取，也可以到mysql官网下载，或者到本地maven库中拷贝一份。

#### Hadoop jar
将Hadoop安装目录下的**share**目录下的所有jar拷贝过来

> cp  ${HADOOP_HOME}/share/hadoop/*/*.jar  . ;
> cp ${HADOOP_HOME}/share/hadoop/*/lib/*.jar  .;

#### 坑
这里有个大坑，oozie server默认使用`tomcat 6.0.41`，而hadoop也有内置的server，如果按照上面两个命令把hadoop依赖的jar包都拷贝过去，有可能出现冲突，这两个server使用的servlet、jsp版本很可能不一样。

这里需要把这几个jar包删除，不要放到libext中

> rm jasper-compiler-5.5.23.jar ;
> rm  jasper-runtime-5.5.23.jar ;
> rm  jsp-api-2.1.jar;

### 配置
前面说到，Oozie会把除了运行日志以外的所有日志全部存放在第三方数据库中，这里我们选用MySQL，所以需要再配置文件中配置MySQL。

#### 创建数据库用户
注意：oozie-site.xml中有个配置项`oozie.service.JPAService.create.db.schema`，默认值为false，表示非自动创建数据库，所以我们需要自己创建oozie数据库。

用root用户登录mysql

> create user 'oozie' identified by 'oozie'; 
> create database oozie;
> grant all privileges on oozie.* to 'oozie'@'%' identified by 'oozie';
> grant all privileges on oozie.* to 'oozie'@'localhost' identified by 'oozie'; 
> flush privileges;

#### 配置
> cd /usr/local/share/applications/oozie-4.3.0/conf; 
> vim oozie-site.xml;

**oozie-site.xml**

    <configuration>
    	<!--mysql作为元数据存放的数据库-->
    	<property>
    		<name>oozie.service.JPAService.jdbc.driver</name>
    		<value>com.mysql.jdbc.Driver</value>
    		<description>
    			JDBC driver class.
    		</description>
    	</property>
    	<property>
    		<name>oozie.service.JPAService.jdbc.url</name>
    		<value>jdbc:mysql://itaojin101:3306/oozie</value>
    		<description>
    			JDBC URL.
    		</description>
    	</property>
    	<property>
    		<name>oozie.service.JPAService.jdbc.username</name>
    		<value>oozie</value>
    		<description>
    			DB user name.
    		</description>
    	</property>
    	<property>
    		<name>oozie.service.JPAService.jdbc.password</name>
    		<value>oozie</value>
    		<description>
    			DB user password.
    			IMPORTANT: if password is emtpy leave a 1 space string, the service trims the value,
    			if empty Configuration assumes it is NULL.
    		</description>
    	</property>
    	<!--设置Hadoop的配置文件的路径-->
    	<property>
    		<name>oozie.service.HadoopAccessorService.hadoop.configurations</name>
    		<value>*=/usr/local/share/applications/hadoop-2.6.0/etc/hadoop</value>
    		<description>
    			Comma separated AUTHORITY=HADOOP_CONF_DIR, where AUTHORITY is the HOST:PORT of
    			the Hadoop service (JobTracker, YARN, HDFS). The wildcard '*' configuration is
    			used when there is no exact match for an authority. The HADOOP_CONF_DIR contains
    			the relevant Hadoop *-site.xml files. If the path is relative is looked within
    			the Oozie configuration directory; though the path can be absolute (i.e. to point
    			to Hadoop client conf/ directories in the local filesystem.
    		</description>
    	</property>
    	<!--设置Spark的配置文件的路径-->
    	<!--
    	<property>
    		<name>oozie.service.SparkConfigurationService.spark.configurations</name>
    		<value>*=/opt/spark-1.4.0-bin-hadoop2.6-hive/conf</value>
    		<description>
    			Comma separated AUTHORITY=SPARK_CONF_DIR, where AUTHORITY is the HOST:PORT of
    			the ResourceManager of a YARN cluster. The wildcard '*' configuration is
    			used when there is no exact match for an authority. The SPARK_CONF_DIR contains
    			the relevant spark-defaults.conf properties file. If the path is relative is looked within
    			the Oozie configuration directory; though the path can be absolute.  This is only used
    			when the Spark master is set to either "yarn-client" or "yarn-cluster".
    		</description>
    	</property>
    	-->
    	<!--
    	设置系统库存放在hdfs中，注意只有在job.properties中将设置oozie.use.system.libpath=true才会引用系统库
    	。注意，下面mycluster是namenode的逻辑名称，根据自己集群的情况进行更改即可-->
    	<property>
    		<name>oozie.service.WorkflowAppService.system.libpath</name>
    		<value>hdfs://mycluster/user/${user.name}/share/lib</value>
    		<description>
    			System library path to use for workflow applications.
    			This path is added to workflow application if their job properties sets
    			the property 'oozie.use.system.libpath' to true.
    		</description>
    	</property>
    </configuration>

重点说一下最后两项配置（不含被注释掉的Spark配置）
* 设置Hadoop的配置文件的路径，需要将Hadoop的配置文件路径改成自己的。
* 该配置项是配置HDFS中的共享jar，这里先明白有这么回事儿，后面步骤会配置。注意，**${user.name}**，这个用户名是Oozie在HDFS中的执行用户，可配置的，这里我采用**root**账户，需要配置Hadoop中的**core-site.xml**文件。

**${HADOOP_HOME}/etc/hadoop/core-site.xml**
新增：
		
    <!-- OOZIE -->
    <property>
    	<name>hadoop.proxyuser.root.hosts</name>
    	<value>itaojin101</value>
    </property>
    <property>
    	<name>hadoop.proxyuser.root.groups</name>
    	<value>root</value>
    </property>

**配置完成后，一定要重启Hadoop使配置生效**

## 打war包
经过以上的配置后就可以将Oozie Server打包运行了。

> cd /usr/local/share/applications/oozie-4.3.0/bin; 
> oozie-setup.sh  prepare-war;

## 初始化Mysql

> ooziedb.sh create -sqlfile oozie.sql -run;

完成此步后会在MySQL的oozie数据库下生成一些表格，这些表格就是用来存放OozieServer运行数据的。
![enter image description here](https://i.imgur.com/VXbatBO.png)

## 安装oozie-sharelib
* 在oozie-4.3.0下有一个**oozie-sharelib-4.3.0.tar.gz**包，将其解压，生成**share**目录。

> tar -zxvf oozie-sharelib-4.3.0.tar.gz;

* 将Mysql的驱动jar添加到`../share/lib/sqoop`下，如果没有该jar，Oozie的sqoop作业将受限。当然，如果导入导出的数据库是oracle就导入oracle的驱动jar。
* 将**share**文件夹上传到HDFS的`/user/root`目录下，因为前面配置的Oozie在HDFS中的操作用户是**root**，所以放在root目录下。
	
> hdfs dfs -put /usr/local/share/applications/oozie-4.3.0/share  /user/root;

## 启动Oozie Server

> oozie-start.sh;

这时在/opt/oozie-4.2.0/oozie-server/webapps目录下多了oozie这个目录。如果缺少jar包或者jar包冲突了可以对$OOZIE_HOME/oozie-server/webapps/oozie/WEB-INF/lib中的进行添加或删除jar包

## 验证
* 使用以下命令验证成功与否

> oozie admin -oozie http://localhost:11000/oozie -status;

如果是**System model:Normal**，表明启动成功，否则失败。

* 访问**http://itaojin101:11000/oozie**
	![enter image description here](https://i.imgur.com/J3zAzHr.png)


## 注意
在运行job之前，最好开启Hadoop的jobhistory。它可以帮助你查看job调度时产生的日志。启动命令：

> mr-jobhistory-daemon.sh start historyserver;

使用jps命令查看是否启动成功，如果出现了 JobHistoryServer进程，表示启动成功。

**在上面修改了Hadoop中的core-site.xml配置文件后还未重启Hadoop，请速速重启。**

## Client安装
Oozie server 安装中已经包括了Oozie client。如果想要在其他机子上也使用Oozie，那么只要在那些机子上安装Oozei的client即可。

* 将`/usr/local/share/applications/oozie-4.3.0`目录下的**oozie-client-4.3.0.tar.gz**远程拷贝到 itaojin102 机器上
	

> scp -r $OOZIE_HOME/oozie-client-4.3.0.tar.gz root@itaojin102:/usr/local/share/applications/;

* 切换到itaojin102机器上，解压
	
> cd /usr/local/share/applications/; 	
> tar -zxvf oozie-client-4.3.0.tar.gz;

* 设置环境变量
	![enter image description here](https://i.imgur.com/N2RIBvr.png)
	**注意**：上面还可以添加一个环境变量，export OOZIE_URL=http://itaojin101:11000/oozie这样在后面的oozie job这个命令中就不需要加 -oozie了

* 就可以通过`oozie job -oozie http://itaojin101:11000/oozie -config ....../job.properties -run`开启一个任务了。
	这里的任务只是占位符，想要真正运行一个任务，往下看。


# Oozie Examples
Oozie自带了一些示例job，我们可以来尝试一下。

## 解压
在`/usr/local/share/applications/oozie-4.3.0`目录下有一个**oozie-examples.tar.gz**包，将其解压到任务目录

> tar -zxvf oozie-examples.tar.gz;

## 组成
进入examples/apps，发现有很多示例，我们以 shell 为例
![enter image description here](https://i.imgur.com/xzUdIMI.png)

在 ../examples/apps/shell 目录下有两个文件
* job.properties：job任务的环境变量等参数
* workflow.xml：具体执行的任务，通过xml的形式描述流程，简称 **hPDL**

**一个完整的任务就是由以上两部分构成**。

## 配置
先编辑 **job.properties**

    <!--hadoop1的配置-->
    <!--
    nameNode=hdfs://localhost:8020
    jobTracker=localhost:8021
    -->
    
    <!--hadoop2的配置，由于我的Hadoop是HA，mycluster是Hadoop的nameservices，rm1和rm2是yarn的两个节点-->
    nameNode=hdfs://mycluster
    jobTracker=rm1,rm2
    
    queueName=default
    examplesRoot=examples
    
    <!--true-job任务使用HDFS中share下的共享jar-->
    oozie.use.system.libpath=true
    
    <!--examples的HDFS目录，需要上传，稍后会讲到-->
    oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/apps/shell
Tips：

> 以上配置的备注和注释是#，但是由于#在mackdown中是标题申明符，所以改用xml的形式。


## 坑
由于oozie的job是通过调用**yarn.resourcemanager.address**配置的端口进行通讯，所以我们需在$HADOOP_HOME/etc/hadoop下对**yarn-site.xml**追加，由于我的yarn集群是HA，所以配置如下：

    <!--RM的applications manager(ASM)端口-->
    <property>
    	<name>yarn.resourcemanager.address.rm1</name>
    	<value>itaojin101:8032</value>
    </property>
    <property>
    	<name>yarn.resourcemanager.address.rm2</name>
    	<value>itaojin102:8032</value>
    </property>

**配置完成后，记得一定要重启yarn集群使配置生效**

## 上传HDFS
将examples这个文件上传到hdfs中的/user/${user.name} 中，我采用的是root这个用户，所以是/user/root

> hdfs dfs -put examples /user/root;

## 第一个job

> oozie job -oozie http://itaojin101:11000/oozie -config
> /usr/local/share/applications/oozie-4.3.0/examples/apps/shell/job.properties
> -run;

![enter image description here](https://i.imgur.com/6DasZv5.png)

**注意**： -oozie 后面跟的是oozie server的地址，-config后面跟的是执行的脚本，除了在hdfs上要有一份examples，在本地也需要一份。这个命令中的/data/installers/examples/apps/shell/job.properties 是本地路径的job.properties，不是hdfs上的。

* 根据上面的job Id ，可以使用下列命令进行查看：

> oozie job -oozie http://itaojin101:11000/oozie -info 0000004-180530210648410-oozie-root-W;

![enter image description here](https://i.imgur.com/KXqDWpD.png)

* 根据上面的job Id ，可以使用下列命令进行日志
	> oozie job -oozie http://itaojin101:11000/oozie -log 0000004-180530210648410-oozie-root-W;

![enter image description here](https://i.imgur.com/GaOu8pI.png)

* 可以访问 **http://itaojin101:11000/oozie** 查看提交的job的情况：
![enter image description here](https://i.imgur.com/V5ggJ3n.png)

webUI可以嫌多信息
1. 任务的整个生命周期都能看到
2. 工作流的有向无环图（DAG）
3. 正常日志和错误日志

等等

**参考网站**：

[https://hadooptutorial.info/oozie-share-lib-does-not-exist-error/](https://hadooptutorial.info/oozie-share-lib-does-not-exist-error/)

[http://blog.csdn.net/teddeyang/article/details/16339533 ](http://blog.csdn.net/teddeyang/article/details/16339533) 

[http://oozie.apache.org/docs/4.2.0/AG_Install.html ](http://oozie.apache.org/docs/4.2.0/AG_Install.html) 

[http://www.cloudera.com/content/cloudera/en/documentation/cdh4/v4-2-0/CDH4-Installation-Guide/cdh4ig_topic_17_6.html](http://www.cloudera.com/content/cloudera/en/documentation/cdh4/v4-2-0/CDH4-Installation-Guide/cdh4ig_topic_17_6.html)

[http://stackoverflow.com/questions/11555344/sqoop-export-fail-through-oozie](http://stackoverflow.com/questions/11555344/sqoop-export-fail-through-oozie)