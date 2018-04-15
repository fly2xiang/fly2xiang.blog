title: "Hive 安装使用"
date: 2018-04-13 22:20:17
tags:
- Hadoop
- 大数据
- HDFS
- Hive
categories: 
- 大数据

## 概述

Hive 提供了一种 SQL 方式编写 MapReduce 程序的方式，可以再进行数据分析任务时使用简单的 SQL 语句进行关联、分组、聚合、排序等查询。
Hive 为大量数据的 OLAP 应用提供解决方案。底层基于 HDFS 存储数据。

## 下载安装

从 [Hive官网](https://hive.apache.org) 下载 tar.gz 包，会自动分配到最合适的镜像站。

```bash
wget http://mirrors.hust.edu.cn/apache/hive/stable-2/apache-hive-2.3.3-bin.tar.gz
tar zxf apache-hive-2.3.3-bin.tar.gz
```

添加到环境变量，这里我习惯建立一个符号链接使用

```bash
ln -s apache-hive-2.3.3-bin hive
export HIVE_HOME=/opt/hive
export PATH=$PATH:$HIVE_HOME/bin
cd hive
```

配置，`conf` 目录下有很多配置模板，拷贝一份进行编辑

```bash
cd conf
cp hive-default.xml.template hive-site.xml
cp hive-log4j2.properties.template hive-log4j2.properties
cp hive-exec-log4j2.properties.template hive-exec-log4j2.properties
```

这里需要修改的是 `hive-site.xml` 文件。在修改配置之前，先了解一下 Hive 的运行方式。
Hive 本身并不会存储数据，数据仍然存放在 HDFS 中，Hive 中定义数据表时其实是定义了 HDFS 中文件的解析方式。表定义了 HDFS 文件中各个列的名称、数据类型。可以将 Hive 表理解为 HDFS 文件的视图。
Hive 将 SQL 语句转换为 MapReduce 程序在 Hadoop 中运行。
一般的，通过数据收集系统将文件存储到 HDFS 存储中，然后通过 Hive 进行计算，最后将结果存入 HBase 或其他数据存储，供在线业务使用。

通常，将 Hive 原数据（包含 Hive 表的定义等）存储到 MySQL 是很好的方案。

先准备 MySQL

```bash
> create database hive_metastore default character set utf8mb4;
> grant all on hive_metastore.* to 'hive'@'%' identified by 'hive';
> grant all on hive_metastore.* to 'hive'@'localhost' identified by 'hive';
```

在 HDFS 上创建相关目录

```
hadoop fs -mkdir -p /tmp
hadoop fs -mkdir -p /user/hive/warehouse
hadoop fs -chmod g+w /tmp
hadoop fs -chmod g+w /user/hive/warehouse
```

其中，`/user/hive/warehouse` 是 `hive-site.xml` 中默认指定的仓库存储位置 `hive.metastore.warehouse.dir`

然后编辑 `hive-site.xml`

```xml
  <!-- MySQL 连接信息 -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/hive_metastore?useSSL=false</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/dbName?ssl=true for postgres database.
    </description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
    <description>Username to use against metastore database</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hive</value>
    <description>password to use against metastore database</description>
  </property>
  
  <!-- 临时目录位置 -->
  <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/tmp/hive/scratchdir</value>
    <description>Local scratch space for Hive jobs</description>
  </property>
  <property>
    <name>hive.downloaded.resources.dir</name>
    <value>/tmp/hive/resources</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
```

为使用 MySQL ，还需要将 MySQL JDBC 驱动复制到 hive/lib 下

```bash
cp mysql-connector-java-5.1.46-bin.jar hive/lib/
```

创建临时文件目录

```bash
mkdir -p /tmp/hive/scratchdir
mkdir -p /tmp/hive/resources
```

启动 MySQL

```bash
/etc/init.d/mysql.server start
```

初始化 Hive Schema

```
hive/bin/schematool -initSchema -dbType mysql
```

初始化后可以登录 MySQL 看到 `hive_metastore` 库中已经创建了一些表

```bash
> use hive_metastore;
> show tables;
+---------------------------+
| Tables_in_hive_metastore  |
+---------------------------+
| AUX_TABLE                 |
| BUCKETING_COLS            |
| CDS                       |
| COLUMNS_V2                |
| COMPACTION_QUEUE          |
| COMPLETED_COMPACTIONS     |
| COMPLETED_TXN_COMPONENTS  |
| DATABASE_PARAMS           |
| DBS                       |
| DB_PRIVS                  |
| DELEGATION_TOKENS         |
| FUNCS                     |
| FUNC_RU                   |
| GLOBAL_PRIVS              |
| HIVE_LOCKS                |
| IDXS                      |
| INDEX_PARAMS              |
| KEY_CONSTRAINTS           |
| MASTER_KEYS               |
| NEXT_COMPACTION_QUEUE_ID  |
| NEXT_LOCK_ID              |
| NEXT_TXN_ID               |
| NOTIFICATION_LOG          |
| NOTIFICATION_SEQUENCE     |
| NUCLEUS_TABLES            |
| PARTITIONS                |
| PARTITION_EVENTS          |
| PARTITION_KEYS            |
| PARTITION_KEY_VALS        |
| PARTITION_PARAMS          |
| PART_COL_PRIVS            |
| PART_COL_STATS            |
| PART_PRIVS                |
| ROLES                     |
| ROLE_MAP                  |
| SDS                       |
| SD_PARAMS                 |
| SEQUENCE_TABLE            |
| SERDES                    |
| SERDE_PARAMS              |
| SKEWED_COL_NAMES          |
| SKEWED_COL_VALUE_LOC_MAP  |
| SKEWED_STRING_LIST        |
| SKEWED_STRING_LIST_VALUES |
| SKEWED_VALUES             |
| SORT_COLS                 |
| TABLE_PARAMS              |
| TAB_COL_STATS             |
| TBLS                      |
| TBL_COL_PRIVS             |
| TBL_PRIVS                 |
| TXNS                      |
| TXN_COMPONENTS            |
| TYPES                     |
| TYPE_FIELDS               |
| VERSION                   |
| WRITE_SET                 |
+---------------------------+
57 rows in set (0.01 sec)
```

执行 Hive 进入到 Hive 命令行，可以创建表，查询表

```bash
hive/bin/hive
```

创建表，这里创建一个用户表，两列，id 和 username，字段间以空格隔开

```bash
hive> create table tbl_user (id int, username string) ROW FORMAT DELIMITED FIELDS TERMINATED BY ' ' STORED AS TEXTFILE;
```

从本地文件导入数据

user_data.txt 内容

```bash
1 Jack
2 Peter
3 Alex
```

```bash
hive> load data local inpath 'user_data.txt' into table tbl_user;
```

导入文件之后，可以使用 `hadoop fs` 命令看到文件内容被存储在 HDFS 中

```bash
hadoop fs -cat /user/hive/warehouse/tbl_user/user_data.txt   
```

查询数据

```bash
hive> SELECT username FROM tbl_user;
OK
Jack
Peter
Alex
Time taken: 9.944 seconds, Fetched: 3 row(s)
```

这里简单查询数据并未被转换为 MapReduce 任务，执行一个统计语句

```bash
hive> SELECT username, COUNT(*) FROM tbl_user GROUP BY username;
```

可以从输出看到这里执行了一个 MapReduce 任务。

## 例子：处理 Nginx 日志

截取几行 Nginx 日志作为数据来源

```
46.161.55.108 - - [08/Apr/2018:14:56:56 +0800] https://60.205.189.113 - "GET /_asterisk/ HTTP/1.1" 404 162 "-" "python-requests/2.18.4" "-"
1.199.93.43 - - [08/Apr/2018:15:11:00 +0800] http://q2.cdn.example.com - "GET / HTTP/1.1" 403 564 "-" "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/35.0.1916.153 Safari/537.36" "171.8.70.8"
5.45.207.60 - - [08/Apr/2018:16:01:01 +0800] https://h5.example.com - "GET /robots.txt HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
5.45.207.60 - - [08/Apr/2018:16:01:03 +0800] https://h5.example.com - "GET /node_modules/postcss-reduce-idents HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
141.8.142.120 - - [08/Apr/2018:16:01:09 +0800] https://h5.example.com - "GET /node_modules/icss-replace-symbols/ HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
141.8.142.120 - - [08/Apr/2018:16:01:10 +0800] https://h5.example.com - "GET /node_modules/babel-helper-replace-supers/lib HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
5.45.207.41 - - [08/Apr/2018:16:01:18 +0800] https://h5.example.com - "GET /node_modules/babel-helper-optimise-call-expression HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
141.8.183.10 - - [08/Apr/2018:16:01:24 +0800] https://h5.example.com - "GET /node_modules/babel-plugin-transform-es2015-modules-commonjs/lib/ HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
37.9.113.74 - - [08/Apr/2018:16:01:57 +0800] https://h5.example.com - "GET /node_modules/_jsesc@0.5.0@jsesc/ HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
141.8.142.21 - - [08/Apr/2018:16:02:00 +0800] https://h5.example.com - "GET /node_modules/webpack-sources/node_modules/ HTTP/1.1" 502 166 "-" "Mozilla/5.0 (compatible; YandexBot/3.0; +http://yandex.com/bots)" "-"
```

从 Nginx 配置可以看到，日志的格式为

```
log_format  main  '$remote_addr - $remote_user [$time_local] $scheme://$host - "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
```

可以看到这里不是简单以字符分割的字段，这里需要以正则方式匹配各个字段。编写正则表达式，分组匹配各个字段。

```
(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) - (.*) \[(.*)\] (.*)\:\/\/(.*) - \"(.*)\" (\d+) (\d+) \"(.*)\" \"(.*)\" \"(.*)\"
```

Hive 表的定义

```bash
hive> CREATE TABLE tbl_access_log
> (
> ip string,
> remote_user string,
> localtime string,
> scheme string,
> host string,
> request string,
> status int,
> len int,
> referer string,
> user_agent string,
> x_forward_for string
> )
> ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe' WITH SERDEPROPERTIES 
> ("input.regex" = "(\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}) - (.*) \\[(.*)\\] (.*)\\:\\/\\/(.*) - \\\"(.*)\\\" (\\d+) (\\d+) \\\"(.*)\\\" \\\"(.*)\\\" \\\"(.*)\\\"")
> STORED AS TEXTFILE;

```

这里注意正则表达式的转义

导入数据

```bash
hive> load data local inpath 'access_last_10.log' into table tbl_access_log;
```

查询数据，可以看到各个字段完整被导入

```bash
hive> SELECT * FROM tbl_access_log;
```

从 HDFS 看到文件存储的是原始的内容

```bash
hadoop fs -cat /user/hive/warehouse/tbl_access_log/access_last_10.log
```

统计查询，这里查询各个 ip 的请求数，数目从大到小排序

```bash
hive> SELECT ip, COUNT(ip) AS c FROM tbl_access_log GROUP BY ip ORDER BY c DESC;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
Query ID = hadoop_20180410072835_2ff17f17-9b0e-4006-9f51-e65701fb9aa8
Total jobs = 2
Launching Job 1 out of 2
Number of reduce tasks not specified. Estimated from input data size: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Starting Job = job_1523242780126_0013, Tracking URL = http://hadoop-host:8088/proxy/application_1523242780126_0013/
Kill Command = /opt/hadoop/bin/hadoop job  -kill job_1523242780126_0013
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1
2018-04-10 07:28:49,045 Stage-1 map = 0%,  reduce = 0%
2018-04-10 07:28:56,412 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 3.11 sec
2018-04-10 07:29:05,889 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 7.58 sec
MapReduce Total cumulative CPU time: 7 seconds 580 msec
Ended Job = job_1523242780126_0013
Launching Job 2 out of 2
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Starting Job = job_1523242780126_0014, Tracking URL = http://hadoop-host:8088/proxy/application_1523242780126_0014/
Kill Command = /opt/hadoop/bin/hadoop job  -kill job_1523242780126_0014
Hadoop job information for Stage-2: number of mappers: 1; number of reducers: 1
2018-04-10 07:29:26,954 Stage-2 map = 0%,  reduce = 0%
2018-04-10 07:29:34,483 Stage-2 map = 100%,  reduce = 0%, Cumulative CPU 3.57 sec
2018-04-10 07:29:44,019 Stage-2 map = 100%,  reduce = 100%, Cumulative CPU 8.89 sec
MapReduce Total cumulative CPU time: 8 seconds 890 msec
Ended Job = job_1523242780126_0014
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 7.58 sec   HDFS Read: 10664 HDFS Write: 342 SUCCESS
Stage-Stage-2: Map: 1  Reduce: 1   Cumulative CPU: 8.89 sec   HDFS Read: 5849 HDFS Write: 301 SUCCESS
Total MapReduce CPU Time Spent: 16 seconds 470 msec
OK
5.45.207.60     2
141.8.142.120   2
5.45.207.41     1
46.161.55.108   1
37.9.113.74     1
141.8.183.10    1
141.8.142.21    1
1.199.93.43     1
Time taken: 70.807 seconds, Fetched: 8 row(s)
```

可以看到这里执行了两个 Job，查询出的统计结果符合预期。

## 总结

Hive 提供了一种很方便的方式进行数据分析，能够在不手动编写 MapReduce 程序时简单通过 SQL 语句进行查询，简化了数据分析的工作。

## 参考链接

[Hive Getting Started](https://cwiki.apache.org/confluence/display/Hive/GettingStarted)
[Hive 导入数据的几种方式](https://www.iteblog.com/archives/949.html)
[Hive 2.1.0 安装](https://kaimingwan.com/post/da-shu-ju/hive2.1.0an-zhuang-hadoop2.7.2huan-jing)
[Hive 与 HBase 的差别是什么，各自适用在什么场景中？](https://www.zhihu.com/question/21677041)



