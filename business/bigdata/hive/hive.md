# Hive是什么
Hive是一个数据仓库基础工具在Hadoop中用来处理结构化数据。它架构在Hadoop之上，总归为大数据，并使得查询和分析方便。

最初，Hive是由Facebook开发，后来由Apache软件基金会开发，并作为进一步将它作为名义下Apache Hive为一个开源项目。它用在好多不同的公司。例如，亚马逊使用它在 Amazon Elastic MapReduce。

# Hive不是

-   一个关系数据库
-   一个设计用于联机事务处理（OLTP）
-   实时查询和行级更新的语言

# Hiver特点

-   它存储架构在一个数据库中并处理数据到HDFS。
-   它是专为OLAP设计。
-   它提供SQL类型语言查询叫HiveQL或HQL。
-   它是熟知，快速，可扩展和可扩展的。

# Hive架构

下面的组件图描绘了Hive的结构：
![enter image description here](https://www.yiibai.com/uploads/allimg/141228/1-14122R10152108.jpg)
该组件图包含不同的单元。下表描述每个单元：

| 单元名称 | 操作 |
|--|--|
| 用户接口/界面 | Hive是一个数据仓库基础工具软件，可以创建用户和HDFS之间互动。用户界面，Hive支持是Hive的Web UI，Hive命令行，HiveHD洞察（在Windows服务器）。 |
| 元存储 | Hive选择各自的数据库服务器，用以储存表，数据库，列模式或元数据表，它们的数据类型和HDFS映射。 |
| HiveQL处理引擎 | HiveQL类似于SQL的查询上Metastore模式信息。这是传统的方式进行MapReduce程序的替代品之一。相反，使用Java编写的MapReduce程序，可以编写为MapReduce工作，并处理它的查询。 |
| 执行引擎 | HiveQL处理引擎和MapReduce的结合部分是由Hive执行引擎。执行引擎处理查询并产生结果和MapReduce的结果一样。它采用MapReduce方法。 |
| HDFS 或 HBASE | Hadoop的分布式文件系统或者HBASE数据存储技术是用于将数据存储到文件系统。 |

# Hive工作原理

下图描述了Hive 和Hadoop之间的工作流程。

![enter image description here](https://www.yiibai.com/uploads/allimg/141228/1-14122R10220b9.jpg)

下表定义Hive和Hadoop框架的交互方式：

| Step No | 操作 |
|--|--|
| 1 | **Execute QueryHive**接口，如命令行或Web UI发送查询驱动程序（任何数据库驱动程序，如JDBC，ODBC等）来执行。 |
| 2 | **Get Plan**在驱动程序帮助下查询编译器，分析查询检查语法和查询计划或查询的要求。 |
| 3 | **Get Metadata**编译器发送元数据请求到Metastore（任何数据库）。 |
| 4 | **Send Metadata**Metastore发送元数据，以编译器的响应。 |
| 5 | **Send Plan**编译器检查要求，并重新发送计划给驱动程序。到此为止，查询解析和编译完成。 |
| 6 | **Execute Plan**驱动程序发送的执行计划到执行引擎。 |
| 7 | **Execute Job**在内部，执行作业的过程是一个MapReduce工作。执行引擎发送作业给JobTracker，在名称节点并把它分配作业到TaskTracker，这是在数据节点。在这里，查询执行MapReduce工作。 |
| 8 | **Metadata Ops**与此同时，在执行时，执行引擎可以通过Metastore执行元数据操作。 |
| 9 | **Fetch Result**执行引擎接收来自数据节点的结果。 |
| 10 | **Send Results**执行引擎发送这些结果值给驱动程序。 |
| 11 | **Send Results**驱动程序将结果发送给Hive接口。 |

# 安装
## 下载并解压
**在这一步，我踩了不少坑，现总结一下：**
* 登录官网，进入下载页面：
![enter image description here](https://i.imgur.com/6tJcQtH.png)
* 点击官网推荐下载链接
* 选择稳定版本进行下载，根据需要选择1.x和2.x，这里我选择了2.3.3版本
![enter image description here](https://i.imgur.com/7rMsNmD.png)
* `cd /usr/local/share/applications/`
* `wget http://apache.cs.utah.edu/hive/stable-2/apache-hive-2.3.3-bin.tar.gz`
* 解压：`tar -zxvf apache-hive-2.3.3-bin.tar.gz

## 配置环境变量
将Hive根目录下的/bin目录加入Path

    vim /etc/profile

![enter image description here](https://i.imgur.com/O36nZ8q.png)

    source /etc/profile

## 将mysql-connector-java.jar复制到Hive的lib目录下
由于我们要采用mysql存储schema数据的方式，所以需要添加mysql链接驱动jar

下载jar有三种方法：
* 到官方直接下载，但是貌似不能选择想要的版本
* 创建一个maven项目，在pom文件中导入想要版本的jar，再到repository中将jar上传至linux服务器
* 由于我在itaojin105上搭建了一套nuxus环境，所以完成第二步后，直接到nuxus拷贝链接，再到linux服务器上wget即可。
[Mysql各版本驱动](http://itaojin105:8081/nexus/content/groups/public/mysql/mysql-connector-java/)

## 配置hive-site.xml
* 进入<hive_home>/conf/目录下，拷贝一份模板

    cp hive-default.xml.template hiv-site.xml
 * 编辑hive-site.xml
	 一共自改三处：
	 1. 搜索并修改Schema数据存放的数据库信息
		![enter image description here](https://i.imgur.com/pA83yeH.png)
	2. 搜索并修改掉**所有**`${system`开头的路径，改成具体路径
	![enter image description here](https://i.imgur.com/yULeisl.png)
	3. 搜索并修改 **hive.server2.enable.doAs**
		![enter image description here](https://i.imgur.com/RQ0ZDPS.png)

## 配置Mysql

* 如果没有安装Mysql，则先安装
* 新增hive用户和hive数据库
	

    create user 'hive' identified by 'hive';
    	create database hive;
* 授权
	

    grant all privileges on hive.* to 'hive'@'%' identified by 'hive';

## 初始化Schema数据
我们知道，我们将hive的schema数据存放在mysql，将真实数据存放在hdfs。
那什么是schema数据呢？简言之，就是**表格信息**。

    schematool -dbType mysql -initSchema

看到成功提示后，访问mysql的hive数据库，能看到新建了跟多表格，这些都是存放schema数据的表格。

## hive
在命令行直接输入**hive**即可进入操作界面，我们可以像操作Mysql一样去操作Hiive，比如我们执行：

    show databases;
    create database test;
    use test;
    create table t (id int, name string, age int);
    insert into t values (1,'zhangsan', 12);
    select * from t;

* 我们再进入存放schema的Mysql数据库，查看我们新建的数据库和表格：
	
	`select * from DBS;`
![enter image description here](https://i.imgur.com/tzWHjkx.png)

    select * from TBLS;
![enter image description here](https://i.imgur.com/cAmwhYP.png)

* 查看数据存放在HDFS中的位置
	

    hdfs dfs -lsr /
	![enter image description here](https://i.imgur.com/BTfMDmy.png)

**hive**命令行已经够方便了，但是它有一个致命缺点：**只能在Hive的安装机器上执行，在其他客户机上不能**。

为了解决以上问题，Hive引入了**hiveserver2**的服务概念，即在服务器上启动一个server进程，端口号默认是**10000**，客户端可以通过连接**hiveserver2**来操作Hive。

## hiveserver2

    hive --service hiveserver2 &
    或者
    hiveserver2 $

后台启动：

> nohup hive --service hiveserver2 &

即可后台启动**hiveserver2**进程
jps可以看到有一个**RunJar**进程，这就是hiveserver2

## beeline
这是一个连接**hiveserver2**服务的客户端

    beeline
    !connect jdbc:hive2://itaojin101:10000/test;
即可进入Hive的命令行界面，功能跟**hive命令行**一模一样

# Java客户端访问
* 导入Jar依赖
	``
	<dependency> 
	 <groupId>org.apache.hive</groupId>  
	 <artifactId>hive-jdbc</artifactId>  
	 <version>2.3.3</version>  
	</dependency>
	``

	**注意：这里有个坑，hive-jdbc的版本必须和服务器的Hive版本对上，不然会报错，切记！**

* java代码
	

    public static void main(String[] args){  
	        Connection connection = null;  
      PreparedStatement pstat = null;  
      ResultSet res = null;  
      
     try{  
            Class.forName("org.apache.hive.jdbc.HiveDriver");  
      connection = DriverManager.getConnection("jdbc:hive2://itaojin101:10000/test");  
      pstat = connection.prepareStatement("select * from t");  
      res = pstat.executeQuery();  
     if(res != null){  
                while (res.next()){  
                    System.out.println(res.getInt(1)+" - "+res.getString(2)+" - "+res.getInt(3));  
      }  
            }  
        }catch (Exception e){  
            e.printStackTrace();  
      }finally {  
            if(connection != null){  
                try {  
                    connection.close();  
      } catch (SQLException e) {  
                    e.printStackTrace();  
      }  
            }  
            if(pstat != null){  
                try {  
                    pstat.close();  
      } catch (SQLException e) {  
                    e.printStackTrace();  
      }  
            }  
            if(res != null){  
                try {  
                    res.close();  
      } catch (SQLException e) {  
                    e.printStackTrace();  
      }  
            }  
        }  
      
    }

## 坑（windows客户端连接Hive）
*  针对问题：

> hadoop2.9.0，运用客户端连接hiveserver2报错 HADOOP_HOME and hadoop.home.dir are unset

  * 文件下载地址
	  [https://pan.baidu.com/s/1H5nba7mHTuv90rGUrJjotA](https://pan.baidu.com/s/1H5nba7mHTuv90rGUrJjotA)
 
* 解决办法：

1. 将文件解压到hadoop的bin目录下  
2. 将hadoop.dll复制到C:\Window\System32下  
3. 添加环境变量HADOOP_HOME，指向hadoop目录  
4. 将%HADOOP_HOME%\bin加入到path里面
5. 重启IDE，或者在代码首行添加：
> System.setProperty("hadoop.home.dir", "E:\\hadoop-2.9.0");


# Hive中的表

## managed table
**托管表** - 删除表时（Mysql上），数据也一并删除（HDFS上）

## external table
**外部表** - 删除表时（Mysql上），数据不被删除（HDFS上）

## 分区表
即表支持分区存储

## 非分区表

## Thinking Time
Tip:

> 本段内容属于超前内容，可以先学习到**分区表**章节后再回来阅读

从上面的分类可以看出：Hive中的表大体分为两大类：**托管表和外部表**，**分区表和非分区表**。那么，如果一张表既既是外部表又是分区表，那会不会有更多的特性呢？

先将两组特性的组合情况编号：
|  | 删分区删数据 | 删分区不删数据 | 不分区 |
|--|--|--|--|
| 删表删数据 | 1 | 2 | 5 |
| 删表不删数据 | 3 | 4 | 6 |

结论：
|  | 分区表 | 非分区表 |
|--|--|--|
| 托管表 | 1 | 5 |
| 外部表 | 4 | 6 |

**为了加强知识吸收，强烈建议通过命令行亲自验证以上结论**

# 数据类型
Hive的数据类型主要分为 **基本数据类型** 和 **集合数据类型** 两大类。

## 基本数据类型
| 数据类型 | 长度 | 例子 |
|--|--|--|
| TINYINT | 1byte有符号整数 | 20 |
| SMALINT | 2byte有符号整数 | 20 |
| INT | 4byte有符号整数 | 20 |
| BIGINT | 8byte有符号整数 | 20 |
| BOOLEAN | 布尔类型，true或者false | true |
| FLOAT | 单精度浮点数 | 3.1415 |
| DOUBLE | 双精度浮点数 | 3.1415 |
| STRING | 字符序列。可以使用单引号或者双引号 | "hello",'world' |
| TIMESTAMP(v0.8.0+) | 整数，浮点数或者字符串 | ①1327882394（Unix新纪元秒）②1327882394.123456789(Unix新纪元秒并跟随有纳秒数)③'2018-06-19 14:42:30.123456789'(JDBC所兼容的java.sql.Timestamp时间格式) |
| BINARY(v0.8.0+) | 字节数组 |  |

## 集合数据类型
| 数据类型 | 描述 | 字面语法示例 |
|--|--|--|
| STRUCT | 可以通过“点”符号访问元素内容。例如某个列的数据类型是STRUCT<first:string,last:string，那么第一个元素可以通过 字段名.first来引用> | struct('John','Doe') |
| MAP | MAP是一组键-值对元组集合，使用数组表示法(例如['key'])可以访问元素。例如某个字段数据类型是MAP，其中键->值对是'first'->'John'和'last'->'Doe'，那么可以通过 字段名['first']来引用第一个元素 | map('first','John','last','Doe') |
| ARRAY | 数组是一组具有相同类型和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，从0开始。 | Array('John','Doe') |

## 举例
1. SQL：

	> create external table t11 (
	>  id int, 
	>  name string, 
	>  info struct<address:string, age:int, birthday:string>, 
	>  education map<string, string>, 
	>  hobby array<string> 
	>  ) 
	>  row format delimited 
	>  fields terminated by ',' 
	>  collection items terminated by '&' 
	>  map keys terminated by ':' 
	>  lines terminated by '\n' 
	>  stored as textfile;

	解读：
	* 定义了基本数据类型的id和name字段，和集合数据类型的info，education和hobby字段
	* 字段之间以 `,` 分割
	* 集合内部元素之间以 `&` 分割
	* MAP的键-值对之间以 `:` 分割
	* 行以 `\n` 分割
	* 数据以文本形式存储

2. 测试数据：

	    将以下测试数据拷贝到一个文件里，这里的文件名称是t11.txt，位于root根目录下
	    
	    1,zhangsan,chengdu&23&1990-11-06,primary school:nonganxiaoxue0&junior:xiangshuizhongxue0&high school:wanerzhong0&university:liaoning0,outdoor&riding&sleep
	    2,lisi,shanghai&23&1991-11-06,primary school:nonganxiaoxue1&junior:xiangshuizhongxue1&high school:wanerzhong1&university:liaoning1,eat&sleep
	    3,wangwu,chongqing&23&1992-11-06,primary school:nonganxiaoxue2&junior:xiangshuizhongxue2&high school:wanerzhong2&university:liaoning2,shopping
	    4,zhaoliu,wuhan&23&1993-11-06,primary school:nonganxiaoxue3&junior:xiangshuizhongxue3&high school:wanerzhong3&university:liaoning3,watching
	    5,heqi,beijing&23&1994-11-06,primary school:nonganxiaoxue4&junior:xiangshuizhongxue4&high school:wanerzhong4&university:liaoning4,thinking
	    6,qiuba,kunming&23&1995-11-06,primary school:nonganxiaoxue5&junior:xiangshuizhongxue5&high school:wanerzhong5&university:liaoning5,busketball

3. 导入数据到Hive

	> load data local inpath '/root/t11.txt' into table t11;

4. 查询数据

	* 全表查询
		> select * from t11;

		![enter image description here](https://i.imgur.com/ybQDdcd.png)

	* 查询STRUCT数据类型的数据

		> select info.address from t11;

		![enter image description here](https://i.imgur.com/92Vb6hM.png)

	* 查询MAP数据类型的数据

		> select education['junior'] from t11;

		![enter image description here](https://i.imgur.com/7cuMqrk.png)

	* 查询ARRAY数据类型的数据

		> select hobby[0] from t11;

		![enter image description here](https://i.imgur.com/lx1yC0X.png)


# Hive命令

## 创建表

    

> CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name  
>     [(col_name data_type [COMMENT col_comment], ...)]  
>     [COMMENT table_comment]  
>     [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]  
>     [CLUSTERED BY (col_name, col_name, ...)  
>     [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]  
>     [ROW FORMAT row_format]  
>     [STORED AS file_format]  
>     [LOCATION hdfs_path]

* **PARTITIONED BY**给表格指定分区，分区的要素可以有多个

* **CREATE TABLE** 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXIST 选项来忽略这个异常

* **EXTERNAL** 关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION）

* **LIKE** 允许用户复制现有的表结构，但是不复制数据

* **COMMENT**可以为表与字段增加描述

* **ROW FORMAT**
    

> DELIMITED 
>     [FIELDS TERMINATED BY char] 
>     [COLLECTION ITEMS TERMINATED BY char]
>     [MAP KEYS TERMINATED BY char]
>     [LINES TERMINATED BY char]
>     | SERDE serde_name
>      [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定 ROW FORMAT 或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，Hive 通过 SerDe 确定表的具体的列的数据。

* **STORED AS**

    

> SEQUENCEFILE
>     | TEXTFILE
>     | RCFILE
>     | INPUTFORMAT input_format_classname OUTPUTFORMAT output_format_classname

如果文件数据是纯文本，可以使用 STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCE 。

举例：

    CREATE external TABLE IF NOT EXISTS t2(
    id int,
    name string,
    age int
    ) COMMENT 'xx' 
    ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' 
    STORED AS TEXTFILE;

### 坑
Hive的Mysql驱动使用的是mysql-connector-java-5.1.17.jar，但是服务器的Mysql版本是5.6.40，当我在hive命令窗口执行"hive>create database mydb2;"时报错：

> FAILED: Execution Error, return code 1 from
> org.apache.hadoop.hive.ql.exec.DDLTask. MetaException(message:You have
> an error in your SQL syntax; check the manual that corresponds to your
> MySQL server version for the right syntax to use near 'OPTION
> SQL_SELECT_LIMIT=DEFAULT' at line 1)

于是到mysql客户端执行

    nysql>SET OPTION SQL_SELECT_LIMIT=DEFAULT

报错

> ERROR 1064 (42000): You have an error in your SQL syntax; check the
> manual that corresponds to your MySQL server version for the right
> syntax to use near 'OPTION SQL_SELECT_LIMIT=DEFAULT' at line 1

终于找到问题所在了，原来是因为低版本的mysql-connector-java-5.1.17.jar在创建数据库的时候发送测试语句`SET OPTION SQL_SELECT_LIMIT=DEFAULT`，上面可以看到mysql5.6的版本已经不在支持该语句，所以会报错，
**解决办法就是将connector版本升级到与MysqlServer一致的版本**。

## load data
虽然Hive已经提供了类sql的 insert语句支持插入数据，但是这种只支持单条插入的语句不能满足大多数生产情况，**load data**命令可以实现导入一个事先格式化好的文件，从而实现批量导入，这种做法有两个好处：
* 批导入提升了效率
* 节省了HDFS中NameNode的管理开销

### 导入本地文件
加载服务器本地的文件

>  load data local inpath '<local_path>' [overwrite] into table <table_name>;

**[overwrite]**：覆盖原有数据，可选
举例：

> load data local inpath '/root/test3.txt' into table t2;

### 导入HDFS中的文件
    

> load data inpath '<local_path>' [overwrite] into table <table_name>;

### Java代码演示

    /** 
     * LoadData测试 
     * * /
     public static void main(String[] args){  
        Connection connection = null;  
        PreparedStatement pstat = null;  
      
    	 try{  
    	        Class.forName("org.apache.hive.jdbc.HiveDriver");  
    			connection = DriverManager.getConnection("jdbc:hive2://itaojin101:10000/test");  
    			  pstat = connection.prepareStatement("load data local inpath '/root/test5.txt' into table t2");  
    			  pstat.execute();  
    			  System.out.println("--success--");  
    	  }catch(Exception ex){  
    	        ex.printStackTrace();  
    	  }finally {  
    	        if(connection != null){  
    	            try {  
    	                connection.close();  
    				  } catch (SQLException e) {  
    				                e.printStackTrace();  
    				  }  
    			  }  
    			  
    		if(pstat != null){  
    			try {  
    			    pstat.close();  
    			} catch (SQLException e) {  
    			    e.printStackTrace();  
    			}  
    	    }  
    	  }  
    }


## 复制表
### 全表复制
在mysql中复制一张表的表结构以及数据：
    

> mysql>create table <table_name1> as select * from <table_name2>;

在Hive中也一样
    
> hive>create table <table_name1> as select * from <table_name2>;

### 只复制表结构
mysql：

    

> mysql>create table <table_name1> like <table_name2>;

Hive:

    

> hive>create table <table_name1> like <table_name2>;

## 一些会触发MR的sql
前面说过，Hive是将HQL转化为Hadoop的MR (MapReduce)来操作HDFS的。更严谨的说，只是一部分HQL会触发MR。

* 普通查询
	

> select * from table_name; 	
> select col_1, col_2 from table_name;
> 	select * from table_name where col_1=value;

以上三种查询均是普通查询，即不带排序和聚合的查询，这种查询语句不会触发MR任务，Hive会立即返回结果，速度比较快。

![enter image description here](https://i.imgur.com/DQ7FA5d.png)

* 排序
	

> select * from table_name order by col_1 DESC;

由于排序的缘故，根据MR架构思想，在HDFS中，先在各个DataNode上查询到满足条件的数据集，然后NameNode统一收集各个NameNode反馈的数据集进行排序，所以会触发MR。
![enter image description here](https://i.imgur.com/jwYgzsx.png)

也可以到Hadoop的MR管理页面查看刚刚执行作业
![enter image description here](https://i.imgur.com/sXNpFYW.png)

* 统计
	

> select count(*) from table_name;

![enter image description here](https://i.imgur.com/25k1gO3.png)


## 分区表
在Mysql中，表分区的概念已然存在，在Hive中得到了完美的支持。

表分区的好处在于：**将同一表中的不同业务场景，或者根据某种算法分组后的数据分开存储，当需要查询时，根据分区规则查询指定分区数据，避免全局扫描搜索，从而提高搜索效率，这也是Hive优化手段之一，对于超大数据这种效率提升越发明显。**

* **创建分区表**
在创建表格时，就应该显示指定该表为分区表，且还要知名分区字段有哪些。

>     CREATE TABLE t3(
>     id int,
>     name string,
>     age int
>     ) 
>     PARTITIONED BY (Year INT, Month INT) 
>     ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;

存在两个分区字段 **Year** 和 **Month**
创建完毕后，通过命令

> desc t3;

查看表结构信息，可以看到分区元信息：
![enter image description here](https://i.imgur.com/TzVitgT.png)

* **添加分区数据**

> alter table t3 add partition (year=2018, month=5);

通过以下命令查看表格的分区信息：

> show partitions t3;

![enter image description here](https://i.imgur.com/qy9S2jT.png)

添加的分区数据，反映到HDFS，是一个个文件夹：

> hive>dfs -lsr / ;

![enter image description here](https://i.imgur.com/ramLihG.png)

* **导入数据**
	还是以 **load data** 命令为例：
	

> load data local inpath '/root/test.txt' into table t3 partition(year=2018,month=5);

![enter image description here](https://i.imgur.com/tXnZbRk.png)

从上图可以得到以下信息：**建表时并没有创建year和month字段，Hive会自动将分区字段作为伪字段存储，在查询时可以用伪字段作为查询条件**。
![enter image description here](https://i.imgur.com/8egUZnP.png)

再看看HDFS中的存储情况：

    hive>dfs -lsr /;

![enter image description here](https://i.imgur.com/xyPGq5c.png)

* **思考？**
	在导入数据之前，如果不添加分区信息，即不执行
	> alter table t3 add partition (year=2018, month=5);
	
	命令，导入数据会报错吗？

我们先来做个实验，直接执行：

    jdbc:hive2://localhost:10000/test> load data local inpath '/root/test.txt' into table t3 partition(year=2018,month=4);

再查看HDFS目录结构，数据导入了！

结论：**分区信息会被自动创建**。

* **删除分区**

> 	alter table t3 drop if exists partition (year=2018, month=4);

**执行完毕后，不仅分区信息被删除，分区下的数据也全部被删除。原因就是该表是managed table类型的表。**

## 桶表
* 定义：简单的理解，根据某个字段的值Hash后，将数据分成多个bucket存储。

* 与分区表的区别：**在HDFS层面，分区表是按照文件夹区分存储位置，桶表则是在同一文件夹下，按照不同编号的文件区分存储位置**。

* 创建：
> CREATE TABLE t4( 
> id int, 	
> name string, 	
> age int 	
> )  
> 	CLUSTERED BY (id) 	INTO 3 BUCKETS 
> ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;

指定分桶的字段是 **id**，桶数为 **3**

* 载入数据
	1. **load data** 命令将对桶表失效
	2. 采用 **insert** 命令载入其他表的数据
		> insert into t4 select id,name,age from t3;
	3. 查看HDFS文件目录		
		> $hive>dfs -lsr /;

![enter image description here](https://i.imgur.com/RbFU4bC.png)
		一共有3个桶，编号分别为：**000000_0**，**000001_0**，**000002_0**。
		再次执行插入语句，且查看HDFS目录：
		![enter image description here](https://i.imgur.com/06tW6Ap.png)

* 桶数的选择
	![enter image description here](https://i.imgur.com/lOgO7uH.png)
	**总结： 桶数太多太少都不好，有一个粗算规则：先评估表的数量级，再除以hdfs的block的大小的2倍，商值就是桶数。**

* 桶表的优点：根据分桶字段查询极快。

## 导入/导出 数据
### 导出数据

> export table <table_name> to '<path_>';
> 
> * <table_name>: 表名
> * <path_>: 导出的HDFS路径

举例：
> export table t3 to '/root/t3.txt';

使用以下命令查看hdfs目录：
> $hive>dfs -lsr /;

![enter image description here](https://i.imgur.com/lJSdzBb.png)

一共导出了两部分数据：

> * _metadata：表结构数据
> * test.txt：文件

### 导入数据

## 排序
### order by
Hive中的order by跟传统的sql语言中的order by作用是一样的，会对查询的结果做一次全局排序，所以说，只有hive的sql中制定了order by所有的数据都会到同一个reducer进行处理（不管有多少map，也不管文件有多少的block只会启动一个reducer）。但是对于大量数据这将会消耗很长的时间去执行。

这里跟传统的sql还有一点区别：如果指定了hive.mapred.mode=strict（默认值是nonstrict）,这时就必须指定limit来限制输出条数，原因是：所有的数据都会在同一个reducer端进行，数据量大的情况下可能不能出结果，那么在这样的严格模式下，必须指定输出的条数。

### sort by
Hive中指定了sort by，那么在每个reducer端都会做排序，也就是说保证了局部有序（每个reducer出来的数据是有序的，但是不能保证所有的数据是有序的，除非只有一个reducer），好处是：执行了局部排序之后可以为接下去的全局排序提高不少的效率（其实就是做一次归并排序就可以做到全局排序了）。

### distribute by和sort by一起使用
ditribute by是控制map的输出在reducer是如何划分的，举个例子，我们有一张表，mid是指这个store所属的商户，money是这个商户的盈利，name是这个store的名字

| mid | money | name |
|--|--|--|
| AA | 15.0 | 商店1 |
| AA | 20.0 | 商店2 |
| BB | 22.0 | 商店3 |
| CC | 44.0 | 商店4 |

> select mid, money, name from store distribute by mid sort by mid asc, money asc;

我们所有的mid相同的数据会被送到同一个reducer去处理，这就是因为指定了distribute by mid，这样的话就可以统计出每个商户中各个商店盈利的排序了（这个肯定是全局有序的，因为相同的商户会放到同一个reducer去处理）。这里需要注意的是distribute by必须要写在sort by之前。

### cluster by
cluster by的功能就是distribute by和sort by相结合，如下2个语句是等价的：
> select mid, money, name from store cluster by mid;
> 等价于
> select mid, money, name from store distribute by mid sort by mid;

如果需要获得与上面的语句一样的效果：
> select mid, money, name from store cluster by mid sort by money;

**注意被cluster by指定的列只能是降序，不能指定asc和desc。**

![enter image description here](https://i.imgur.com/AWCIeVm.png)

## 作业参数
* 设置reducetask的字节数
> $hive>set hive.exec.reducers.bytes.per.reducer=xxx

* 设置reduce task的最大任务数
> $hive>set hive.exec.reducers.max=0

* 设置reducetask个数
> $hive>set mapreduce.job.reduces=0



