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

