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
