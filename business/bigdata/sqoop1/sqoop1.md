# 前言
笔者书写本文章的时间是2018-06-12，当前的Sqoop分为两个主流版本，1.x和2.x，1.x最新版本是[1.4.7](http://mirror.bit.edu.cn/apache/sqoop/1.4.7/)，2.x最新版本是[1.99.7](http://mirror.bit.edu.cn/apache/sqoop/1.99.7/)。本文章主要讲解1.4.7版本的Sqoop相关概念以及安装使用。

# 概念
Sqoop是Hadoop生态体系中的一员，主要功能是实现**关系型数据库**和**HDFS、Hive、HBase**等组件间的数据互导，该组件在Hadoop生态中所处的位置是**数据采集环节**。

## Sqoop1和Sqoop2的对比
* Sqoop1解压即用，非常轻量级，且相比于Sqoop2的服务化架构，运行更加稳定；缺点就是复用性差，每一个job都需要从头到尾从新编写，即便是编写了服用的Job文件也显得不够灵活。
* Sqoop2采用服务化架构，server直接跟Hadoop的Map Task交互，Sqoop客户端直接跟server交互，且将一个完整的Job拆分成多个连接和job模块，每个模块间可以相互复用；缺点是不够稳定，就我个人使用感受而言至少是的，且服务化架构不够轻量，Cloudera公司在将来的CDH 6版本中将不再支持Sqoop2.

## Sqoop1架构
![enter image description here](https://i.imgur.com/gRws8bt.jpg)

这是一张网上流传较广的Sqoop1架构图，架构很简单，一个Sqoop客户端直接跟Hadoop的Map Task交互，从而为**关系型数据库**和**HDFS/Hive/HBase**之间搭起了一座交互的桥梁。

这里不再介绍Sqoop2的架构和原理，可以通过以下链接参阅一片写的不错的博客：[https://blog.csdn.net/gamer_gyt/article/details/55225700](https://blog.csdn.net/gamer_gyt/article/details/55225700)

## 官方学习文档
[http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html)

# 安装

## 环境说明
默认已经安装了：
* Hadoop集群2.x，包括HDFS和Yarn
* Hive，版本1.x和2.x皆可
* HBase
* Zookeeper
* 测试用的MySQL数据库

我将在**itaojin106**机器上安装Sqoop，当前该机器上的环境如下：
|  |  |  |  |  |  |
|--|--|--|--|--|--|
| **节点** | DataNode | NodeManager | QuorumPeerMain | RunJar | HRegionServer |


## 下载
[1.4.7下载链接](http://mirror.bit.edu.cn/apache/sqoop/1.4.7/)
![enter image description here](https://i.imgur.com/G7G6HgC.png)

发现有两个版本，其中一个后缀Hadoop-2.6.0，如果搭建的Hadoop是2.6.0版本，下载这个将拥有最好的兼容性，如果不是，可以下载第二个不带Hadoop后缀的版本。

> cd /usr/local/share/applications 

> wget http://mirror.bit.edu.cn/apache/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz

## 解压

> tar -zxvf sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz

## 设置环境变量

> vim /etc/profile


> ![enter image description here](https://i.imgur.com/KM1g5OS.png)
> 

> source /etc/profile

验证：

> echo $SQOOP_HOME

能打印出刚才设置的路径则说明配置成功

## 配置
复制$SQOOP_HOME/conf/sqoop-env-template.sh为sqoop-env.sh

> cd $SQOOP_HOME/conf

> cp sqoop-env-template.sh sqoop-env.sh

## 导入数据库驱动jar
由于我们的业务中关系型数据库以MySQL为主，所以这里以MySQL举例：

下载MySQL驱动jar，并放入$SQOOP_HOME/lib下

> cd $SQOOP_HOME/lib

> wget http://192.168.1.105:8081/nexus/content/groups/public/mysql/mysql-connector-java/5.1.38/mysql-connector-java-5.1.38.jar

保持默认配置不用修改，最好要保证每一项配置都能找到相应的目录，即保证安装的机器上都要安装各个组件，由于itaojin106满足条件，故而选之。

## 除去警告

> cd $SQOOP_HOME/bin

> vim configure-sqoop

注释掉如下代码，不然每当运行**sqoop**命令式都会有警告：

    ## Moved to be a runtime check in sqoop.
    #if [ ! -d "${HCAT_HOME}" ]; then
    #  echo "Warning: $HCAT_HOME does not exist! HCatalog jobs will fail."
    #  echo 'Please set $HCAT_HOME to the root of your HCatalog installation.'
    #fi
    
    #if [ ! -d "${ACCUMULO_HOME}" ]; then
    #  echo "Warning: $ACCUMULO_HOME does not exist! Accumulo imports will fail."
    #  echo 'Please set $ACCUMULO_HOME to the root of your Accumulo installation.'
    #fi

## 同步Hive的jackson-*.jar
在执行命令的过程中，遇到了坑，原因就是Hive中lib下的jackson-*.jar和Sqoop的lib下的版本不一致，需要将Hive中的jackson-*.jar复制到Sqoop中

如果Hive和Sqoop中的jackson版本不一致，会报如下的错误

        18/06/12 11:09:33 INFO ql.Driver: Starting task [Stage-0:DDL] in serial mode
    18/06/12 11:09:34 ERROR exec.DDLTask: java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.ObjectMapper.readerFor(Ljava/lang/Class;)Lcom/fasterxml/jackson/databind/ObjectReader;
    	at org.apache.hadoop.hive.common.StatsSetupConst$ColumnStatsAccurate.<clinit>(StatsSetupConst.java:165)
    	at org.apache.hadoop.hive.common.StatsSetupConst.parseStatsAcc(StatsSetupConst.java:297)
    	at org.apache.hadoop.hive.common.StatsSetupConst.setBasicStatsState(StatsSetupConst.java:230)
    	at org.apache.hadoop.hive.common.StatsSetupConst.setBasicStatsStateForCreateTable(StatsSetupConst.java:292)
    	at org.apache.hadoop.hive.ql.plan.CreateTableDesc.toTable(CreateTableDesc.java:839)
    	at org.apache.hadoop.hive.ql.exec.DDLTask.createTable(DDLTask.java:4321)
    	at org.apache.hadoop.hive.ql.exec.DDLTask.execute(DDLTask.java:354)
    	at org.apache.hadoop.hive.ql.exec.Task.executeTask(Task.java:199)
    	at org.apache.hadoop.hive.ql.exec.TaskRunner.runSequential(TaskRunner.java:100)
    	at org.apache.hadoop.hive.ql.Driver.launchTask(Driver.java:2183)
    	at org.apache.hadoop.hive.ql.Driver.execute(Driver.java:1839)
    	at org.apache.hadoop.hive.ql.Driver.runInternal(Driver.java:1526)
    	at org.apache.hadoop.hive.ql.Driver.run(Driver.java:1237)
    	at org.apache.hadoop.hive.ql.Driver.run(Driver.java:1227)
    	at org.apache.hadoop.hive.cli.CliDriver.processLocalCmd(CliDriver.java:233)
    	at org.apache.hadoop.hive.cli.CliDriver.processCmd(CliDriver.java:184)
    	at org.apache.hadoop.hive.cli.CliDriver.processLine(CliDriver.java:403)
    	at org.apache.hadoop.hive.cli.CliDriver.processLine(CliDriver.java:336)
    	at org.apache.hadoop.hive.cli.CliDriver.processReader(CliDriver.java:474)
    	at org.apache.hadoop.hive.cli.CliDriver.processFile(CliDriver.java:490)
    	at org.apache.hadoop.hive.cli.CliDriver.executeDriver(CliDriver.java:793)
    	at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:759)
    	at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:686)
    	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    	at java.lang.reflect.Method.invoke(Method.java:498)
    	at org.apache.sqoop.hive.HiveImport.executeScript(HiveImport.java:331)
    	at org.apache.sqoop.hive.HiveImport.importTable(HiveImport.java:241)
    	at org.apache.sqoop.tool.ImportTool.importTable(ImportTool.java:537)
    	at org.apache.sqoop.tool.ImportTool.run(ImportTool.java:628)
    	at org.apache.sqoop.Sqoop.run(Sqoop.java:147)
    	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
    	at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:183)
    	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:234)
    	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:243)
    	at org.apache.sqoop.Sqoop.main(Sqoop.java:252)
    
    FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. com.fasterxml.jackson.databind.ObjectMapper.readerFor(Ljava/lang/Class;)Lcom/fasterxml/jackson/databind/ObjectReader;
    18/06/12 11:09:34 ERROR ql.Driver: FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. com.fasterxml.jackson.databind.ObjectMapper.readerFor(Ljava/lang/Class;)Lcom/fasterxml/jackson/databind/ObjectReader;
    18/06/12 11:09:34 INFO ql.Driver: Completed executing command(queryId=root_20180612110925_f1db6aca-208a-4cb9-b7c6-e584079f1a5c); Time taken: 1.238 seconds
    18/06/12 11:09:34 INFO conf.HiveConf: Using the default value passed in for log id: 85c91a62-b6d3-408f-a4af-b173c719000e
    18/06/12 11:09:34 INFO session.SessionState: Resetting thread name to  main
    18/06/12 11:09:34 INFO conf.HiveConf: Using the default value passed in for log id: 85c91a62-b6d3-408f-a4af-b173c719000e
    18/06/12 11:09:34 INFO session.SessionState: Deleted directory: /tmp/hive/root/85c91a62-b6d3-408f-a4af-b173c719000e on fs with scheme hdfs
    18/06/12 11:09:34 INFO session.SessionState: Deleted directory: /tmp/root/85c91a62-b6d3-408f-a4af-b173c719000e on fs with scheme file
    18/06/12 11:09:34 INFO hive.metastore: Closed a connection to metastore, current connections: 0
    18/06/12 11:09:34 ERROR tool.ImportTool: Import failed: java.io.IOException: Hive CliDriver exited with status=1
    	at org.apache.sqoop.hive.HiveImport.executeScript(HiveImport.java:355)
    	at org.apache.sqoop.hive.HiveImport.importTable(HiveImport.java:241)
    	at org.apache.sqoop.tool.ImportTool.importTable(ImportTool.java:537)
    	at org.apache.sqoop.tool.ImportTool.run(ImportTool.java:628)
    	at org.apache.sqoop.Sqoop.run(Sqoop.java:147)
    	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
    	at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:183)
    	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:234)
    	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:243)
    	at org.apache.sqoop.Sqoop.main(Sqoop.java:252)

先备份$SQOOP_HOME/lib下的jackson-*.jar

> cd $SQOOP_HOME

> mkdir jackson-bak

> mv jackson-*.jar jackson-bak/

拷贝$HIVE_HOME/lib下的jackson-*.jar到$SQOOP_HOME/lib下

> cp $HIVE_HOME/lib/jackson-*.jar .

## 测试

> sqoop list-databases --connect jdbc:mysql://localhost:3306 --username root --password root

运行结果如下：

![enter image description here](https://i.imgur.com/mzozn6M.png)

至此，安装完毕！！！

# 语法

Sqoop有许多组工具，罗列如下：
| Sqoop工具 |
|--|
| [`sqoop-import`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_import_literal) |
| [`sqoop-import-all-tables`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_import_all_tables_literal) |
| [`sqoop-import-mainframe`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_import_mainframe_literal) |
| [`sqoop-export`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_export_literal) |
| [`validation`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#validation) |
| [`sqoop-job`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_job_literal) |
| [`sqoop-metastore`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_metastore_literal) |
| [`sqoop-merge`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_merge_literal) |
| [`sqoop-codegen`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_codegen_literal) |
| [`sqoop-create-hive-table`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_create_hive_table_literal) |
| [`sqoop-eval`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_eval_literal) |
| [`sqoop-list-databases`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_list_databases_literal) |
| [`sqoop-list-tables`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_list_tables_literal) |
| [`sqoop-help`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_help_literal) |
| [`sqoop-version`](http://sqoop.apache.org/docs/1.4.7/SqoopUserGuide.html#_literal_sqoop_version_literal) |

## sqoop-import
Sqoop最重要的功能，也是业务中用得最多的情形是 **将关系型数据库中的数据导入HDFS/Hive/HBase**。

### 连接数据库服务器 
| 参数 | 描述 |
|--|--|
| `--connect <jdbc-uri>` | 指定JDBC连接地址(`jdbc:mysql://<ip>:<port>[/<schema>]`) |
| `--connection-manager <class-name>` | 指定连接管理器主类 |
| `--driver <class-name>` | 指定JDBC驱动程序类 |
| `--hadoop-mapred-home <dir>` | 覆盖配置中的 $HADOOP_MAPRED_HOME |
| `--username <username>` | 设置数据库用户名 |
| `--password <password>` | 设置数据库密码（明文） |
| `-P` | 从控制台输入流读取数据库密码 |
| `--password-file` | 从一个文件中读取密码 |
| `--verbose` | 打印更加详细的信息 |
| `--connection-param-file <filename>` | 从一个文件中读取数据库连接信息 |
| `--relaxed-isolation` | 设置事务隔离级别 |

* 测试与数据库的连通性
> sqoop list-databases --connect jdbc:mysql://localhost:3306 --username root --password root

* 以下命令等同于上面的命令：
> sqoop list-databases --connect jdbc:mysql://localhost:3306 --username root -P

只不过需要再控制台手动输入密码，这样就提高了安全性。

### 有选择性的导入数据
| 参数 | 描述 |
|--|--|
| `--append` | HDFS中，往已存在的文件中追加内容 |
| `--as-avrodatafile` | 以Avro的格式导入 |
| `--as-sequencefile` | 以序列化文件的格式导入 |
| `--as-textfile` | 以文本的格式导入 |
| `--as-parquetfile` | Imports data to Parquet Files |
| `--boundary-query <statement>` | 自定义边界查询SQL，默认是以主键的最大最小值作为边界 |
| `--columns <col,col,col…>` | 设定需要导入的列名 |
| `--delete-target-dir` | HDFS中，如果导入目录存在则删除 |
| `--direct` | 采用mysqldump工具替代JDBC直连 |
| `--fetch-size <n>` | 从数据库中读取的条目的数量 |
| `--inline-lob-limit <n>` | Set the maximum size for an inline LOB |
| `-m,--num-mappers <n>` | 并行运行的Map Task的数量 |
| `-e,--query <statement>` | 自定义查询语句 |
| `--split-by <column-name>` | 用于分割的列名，默认是主键，该配置不能与`--autoreset-to-one-mapper`共同使用 |
| `--split-limit <n>` | 每个分割大小的上限，这只适用于Integer和Date类型（date, timestamp）字段，对于date或timestamp字段，它是以秒为单位计算的 |
| `--autoreset-to-one-mapper` | Import should use one mapper if a table has no primary key and no split-by column is provided. Cannot be used with  `--split-by <col>`  option. |
| `--table <table-name>` | 需要被导出的MySQL中的表名 |
| `--target-dir <dir>` | HDFS中的输出文件目录 |
| `--temporary-rootdir <dir>` | 设置导入HDFS时存放文件的临时目录，会覆盖默认的“_sqoop”目录 |
| `--warehouse-dir <dir>` | HDFS parent for table destination |
| `--where <where clause>` | 设置查询条件，作为数据导入的查询条件 |
| `-z,--compress` | 开启压缩功能 |
| `--compression-codec <c>` | 再开启压缩功能前提下，设置压缩文件的格式，默认“gzip” |
| `--null-string <null-string>` | The string to be written for a null value for string columns |
| ``--null-non-string <null-string>`` | The string to be written for a null value for non-string columns |

* 一个导入的例子：
> sqoop import --connect jdbc:mysql://localhost:3306/itaojin --username root \\
> --password root --table t --delete-target-dir --target-dir /user/sqoop1 -m 2 \\
> --direct  --columns "name,age,address" --where "age > 24"

### 灵活的查询导入
| 参数 | 描述 |
|--|--|
| `--query` | 自定义查询语句，灵活易用 |

* 举例
	> sqoop import --connect jdbc:mysql://localhost:3306/itaojin \\
	> --username root --password root  \\
	> --query 'select * from t where $CONDITIONS'  \\ 	
	> --split-by id --target-dir /user/sqoop1/temp_dir

* 使用“--query”导入数据时必须遵循两个条件：
		1. 必须指定 `--target-dir`
		2. 必须指定 `--split-by` 或 `-m 1`
* 注意：`--query`后如果是双引号(" ")，则where条件后跟随 `\$CONDITIONS`；如果是单引号(' ')，则where后面跟随 `$CONDITIONS`

### 并行控制
即控制Map Task并行计算的机制

| 参数 | 描述 |
|--|--|
| -m, --num-mappers | 设置Map Task的并行数量 |

* 默认情况下，该参数被设置为 4
* 增加Map Task的数量能显著提高计算效率，但是也不是数量越大越好，一般情况下，此数量不要超过Hadoop集群中的NodeManager节点数。
* Sqoop有一套切割工作量的策略。默认情况下，Sqoop会识别数据库中表的主键作为**分割字段（splitting column）**，与此同时，Sqoop会通过**SELECT MAX(split-column), MIN(split-column) FROM _TABLE**查询语句查询出最大和最小值，以此作为数据集合的边界，然后根据**--num-mappers**设置的数量均分边界范围，以此来达到将总数据平均分配到每个Map Task上。举个例子：有张表主键为id，且最小最大值分别为0,1000，Sqoop默认使用4个Map Task，Sqoop会分别启动四个进程，且分别执行SQL查询：
			
	> SELECT * FROM _TABLE WHERE id >=0 AND id<250;
	> 	SELECT * FROM _TABLE WHERE id >=250 AND id<500;
	> 	SELECT * FROM _TABLE WHERE id >=500 AND id<750;
	> 	SELECT * FROM _TABLE WHERE id >=750 AND id<1000;

	从而确定每个Map Task的数据集合。
* 从上面的描述中，会发现有两个地方值得注意：
			1. **分割字段必须是num型或者date型，因为Sqoop会依据此字段做最大最小边界查询。**
			2. **分割字段的选取，最好遵循其变化范围是均匀分布的准则，否则导致每个Map Task分配的任务不均导致整体性能下降（木桶原理）。**
If the actual values for the primary key are not uniformly distributed across its range, then this can result in unbalanced tasks.
* 当表格的主键的取值范围分布不均时，可以通过`--split-by`指令重新指定分割字段。
* User can override the `--num-mapers` by using `--split-limit` option. Using the `--split-limit` parameter places a limit on the size of the split section created. If the size of the split created is larger than the size specified in this parameter, then the splits would be resized to fit within this limit, and the number of splits will change according to that.This affects actual number of mappers. If size of a split calculated based on provided `--num-mappers` parameter exceeds `--split-limit` parameter then actual number of mappers will be increased.If the value specified in `--split-limit` parameter is 0 or negative, the parameter will be ignored altogether and the split size will be calculated according to the number of mappers.
* 如果表格没有主键，且也没有定义分割字段，	可以将Map Task的数量设置成1即可，`--num-mappers 1` 或 `-m 1`，否则会报错。

### 分布式缓存
当Sqoop每次开启job时，都会将`$SQOOP_HOME/lib`文件夹拷贝到job的缓存中。当Sqoop被Oozie调度时，这种拷贝是很没有必要的，因为Oozie在分布式缓存中拥有包含了Sqoop所有依赖的且属于自己的Sqoop共享lib。当oozie集群中的每一个节点**首次**执行Sqoop job时会将Sqoop共享lib在每个工作节点上实现本地化缓存，当以后的job再次执行，直接复用本地缓存jar。**所以，当Sqoop被Oozie调度时，使用 `--skip-dist-cache`指令使得Oozie跳过Sqoop拷贝jar至job缓存的过程，从而可以节省大量的 I/O**

官方原文如下：
> Sqoop will copy the jars in $SQOOP_HOME/lib folder to job cache every time when start a Sqoop job. When launched by Oozie this is unnecessary since Oozie use its own Sqoop share lib which keeps Sqoop dependencies in the distributed cache. Oozie will do the localization on each worker node for the Sqoop dependencies only once during the first Sqoop job and reuse the jars on worker node for subsquencial jobs. Using option `--skip-dist-cache` in Sqoop command when launched by Oozie will skip the step which Sqoop copies its dependencies to job cache and save massive I/O.

### direct模式
| 参数 | 描述 |
|--|--|
| --direct | 开启direct模式，Sqoop将采用mysqldump工具导出数据 |


默认情况下，采用JDBC直连的方式为数据导入提供数据通道。

很多数据库都有比较高效的数据导入/导出工具，例如MySQL数据库的`mysqldump`。只需提供指令`--direct`就能开启**direct模式**，这种模式下的数据通道在效率上高于JDBC直连方式。

当采用direct模式时，还可以指定额外参数，通过指令 `--` 指定额外参数，举例：

> sqoop import --connect <jdbc_url> --username <username> \\
> --password <password> --table <table_name> --direct \\
> -- --default-character-set=latin1 

### 事务隔离级别
| 参数 | 描述 |
|--|--|
| `--relaxed-isolation` | Sqoop开启**读未提交**事务隔离级别 |

事务隔离级别分为四种：

* **Read uncommitted**
	读未提交，顾名思义，就是一个事务可以读取另一个未提交事务的数据。
	
	事例：老板要给程序员发工资，程序员的工资是3.6万/月。但是发工资时老板不小心按错了数字，按成3.9万/月，该钱已经打到程序员的户口，但是事务还没有提交，就在这时，程序员去查看自己这个月的工资，发现比往常多了3千元，以为涨工资了非常高兴。但是老板及时发现了不对，马上回滚差点就提交了的事务，将数字改成3.6万再提交。
	
	分析：实际程序员这个月的工资还是3.6万，但是程序员看到的是3.9万。他看到的是老板还没提交事务时的数据。这就是脏读。
	那怎么解决脏读呢？Read committed！读提交，能解决脏读问题。
* **Read committed**
	读提交，顾名思义，就是一个事务要等另一个事务提交后才能读取数据。
	事例：程序员拿着信用卡去享受生活（卡里当然是只有3.6万），当他埋单时（程序员事务开启），收费系统事先检测到他的卡里有3.6万，就在这个时候！！程序员的妻子要把钱全部转出充当家用，并提交。当收费系统准备扣款时，再检测卡里的金额，发现已经没钱了（第二次检测金额当然要等待妻子转出金额事务提交完）。程序员就会很郁闷，明明卡里是有钱的…
	分析：这就是读提交，若有事务对数据进行更新（UPDATE）操作时，读操作事务要等待这个更新操作事务提交后才能读取数据，可以解决脏读问题。但在这个事例中，出现了一个事务范围内两个相同的查询却返回了不同数据，这就是不可重复读。
	那怎么解决可能的不可重复读问题？Repeatable read ！
* **Repeatable read**
	重复读，就是在开始读取数据（事务开启）时，不再允许修改操作
	事例：程序员拿着信用卡去享受生活（卡里当然是只有3.6万），当他埋单时（事务开启，不允许其他事务的UPDATE修改操作），收费系统事先检测到他的卡里有3.6万。这个时候他的妻子不能转出金额了。接下来收费系统就可以扣款了。
	分析：重复读可以解决不可重复读问题。写到这里，应该明白的一点就是，不可重复读对应的是修改，即UPDATE操作。但是可能还会有幻读问题。因为幻读问题对应的是插入INSERT操作，而不是UPDATE操作。
原文链接：[https://blog.csdn.net/qq_33290787/article/details/51924963](https://blog.csdn.net/qq_33290787/article/details/51924963)

言归正传，默认情况下，Sqoop采用**读提交（read committed）**事务隔离级别，在工作流的ETL整个过程中，这或许不是一个完全使用的隔离级别。通过指令 `--relaxed-isolation` 能让Sqoop采用**读未提交**的事务隔离级别。

**Tips**： **读未提交**事务隔离级别不适用与全部的数据库，比如Oracle，所以在指定`--relaxed-isolation`指令时应该根据业务中采用的数据库而定。

### 控制字段类型映射

| 参数 | 描述 |
|--|--|
| `--map-column-java <mapping>` | 覆盖默认的从SQL到Java的映射规则，字段之间以**逗号**隔离 |
| `--map-column-hive <mapping>` | 覆盖默认的从SQL到Hive的映射规则，字段之间以**逗号**隔离 |


什么是字段类型映射？当Sqoop将数据从关系型数据导出数据时，会将数据库中的每个字段映射到Java或者Hive中的字段，这时就存在字段类型转换的问题。

Sqoop被预先配置为将大多数SQL类型映射到适当的Java或Hive代表，但是默认的映射机制可能不适用于全部的场景，这时就可以通过指令 `--map-column-java`（改变数据映射到Java的规则）和 `--map-column-hive` （改变数据映射到Hive的规则）

举例：
> sqoop import ... --map-column-java id=String,value=Integer

**Tips**:
* 当指定 `--map-column-hive <mapping>` 时，字段和类型必须要经过**URL编码**后才能使用，举例，用 **DECIMAL(1%2C%201)** 替代 **DECIMAL(1, 1)**）
* Sqoop将直接抛出类型映射的异常。

### Schema name handling
当Sqoop从企业应用导入数据时，表名和字段名可能含有Java或者Avro/Parquet不识别的特殊字符。为了解决这个问题，Sqoop将这些特殊字符转换成 `_` ，且任何以 `_` 开头的名称都会转为 `__`，例如，`_AVRO`被转换为`__AVRO`

### 增量导入
| 参数 | 描述 |
|--|--|
| `--check-column (col)` | 指定被检查列，该列不能是CHAR/NCHAR/VARCHAR/VARNCHAR/ LONGVARCHAR/LONGNVARCHAR类型，最好用num或者date类型 |
| `--incremental (mode)` | 指定Sqoop确定新行的模式，取值为 `append` 和 `lastmodified` |
| `--last-value (value)` | 设定被检查列在历史数据集合中的最大值，用于边界作用 |

两种增量模式：
* append ： 主要用于导入相比于历史数据是新增的数据
* lastmodified：主要用于更新导入对历史数据的更改。值得注意的是，使用该模式时，必须加上指令 `--append` 或者 `--merge-key <column>`，否则会报错：
`ERROR tool.ImportTool: Import failed: --merge-key or --append is required when using --incremental lastmodified and the output directory exists.`

举例：

> sqoop import --connect jdbc:mysql://localhost:3306/test \\
> --username test --password test --table customertest  \\
> --target-dir /user/sqoop1/test -m 1 --direct \\
> --check-column id \\
> --incremental append \\
> --last-value 5

> sqoop import --connect jdbc:mysql://localhost:3306/test \\
> --username test --password test --table customertest \\
> --target-dir /user/sqoop1/test -m 1 --direct \\
> --check-column last_mod \\
> --incremental lastmodified \\
> --last-value '2018-06-13 15:04:28'  \\
> --append

### 文件格式
| 参数 | 描述 |
|--|--|
| `-z, --compress` | 将数据压缩成gzip格式 |
| `--compression-codec` | 执行Hadoop任意的压缩格式 |

导入的文件有两种格式：**分割文本** 和 **序列化文件**
默认情况下，文件导入采用分割文本的格式，也可以显式的指定导入文件的格式为分割文本：`--as-textfile` 。

序列化文件以二进制格式被存储，相比分割文本，序列化文件具有更高读取性能。

### 导入数据到Hive
| 参数 | 描述 |
|--|--|
| `--hive-home <dir>` | 覆盖 `$HIVE_HOME` |
| `--hive-import` | 导入数据至Hive |
| `--hive-overwrite` | 覆盖Hive中已存在Table中的数据 |
| `--create-hive-table` | 设置后，如果Hive中目标Table已经存在，则job将失败，默认该属性未开启 |
| `--hive-table <table-name>` | 导入Hive中的Table名 |
| `--hive-drop-import-delims` | 当导入数据时，将字段中的`_\n_, _\r_, and _\01_`删除 |
| `--hive-delims-replacement <replace_string>` | 当导入数据时，使用给定的字符串替换`_\n_, _\r_, and _\01_`等字符 |
| `--hive-partition-key <key>` | 指定用于分片的字段名称 |
| `--hive-partition-value <v>` | 本次导入的分片值 |
| `--map-column-hive <map>` | 覆盖默认的SQL与Hive字段类型转换规则，在值中如果包含逗号，则值必须进行URL编码，举例：DECIMAL(1, 1)改写成DECIMAL(1%2C%201) |

* 将关系型数据库数据导入Hive分两步：
		1. 先将数据从关系型数据库导入HDFS
		2. 再将数据从HDFS导入Hive

* 如果关系型数据库中每行数据的某些字段包含Hive默认的**行分隔符 (`\n`和`\r`)  或者 字段分隔符(`\01` characters)**，可以使用`--hive-drop-import-delims`指令来删除这些字符，也可以使用指令`--hive-delims-replacement`用用户自定义字符替换掉特殊字符。
* 如果关系型数据库中字段值为NULL，则在Hive被映射为`null`。在Hive中，用字符串`\N`标示NULL值，`\\N`标示空字符串.

> 	$ sqoop import  ... --null-string '\\N' --null-non-string '\\N'

* 如果不适用指令`--hive-table`，则Sqoop将关系型数据库表名直接用于Hive表名，如果指定，则可以生成用户指定的表名。
* Hive can put data into partitions for more efficient query performance. You can tell a Sqoop job to import data for Hive into a particular partition by specifying the `--hive-partition-key` and `--hive-partition-value` arguments. The partition value must be a string.
* 例子：
	

> sqoop import  \\ 	
> --hive-import \\ 	
> --hive-table hive_customer_test \\
> 	--connect jdbc:mysql://localhost:3306/local \\ 	
> --username local \\
> 	--password local \\ 	
> --table customertest \\ 	
> --target-dir /user/sqoop1/test \\ 	
> -m 1 \\ 	
> --direct

	
