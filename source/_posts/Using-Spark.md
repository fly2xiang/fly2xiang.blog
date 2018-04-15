title: "Spark 2.3.0 安装使用入门"
date: 2018-04-14 21:20:17
tags:
- Hadoop
- 大数据
- Spark
categories: 
- 大数据

---

## 概述

Spark 可以认为是改进版的 MapReduce，改进了 MapReduce 存在的以下问题

* 调度慢，启动耗时，由于 MapReduce 使用进程级的调度，相比 Spark 线程级调度，启动较慢；
* 计算慢，每一步都要保存中间结果到磁盘，相比 Spark 使用内存缓存中间结果较为耗时；
* 使用复杂，只提供 Map/Reduce 两种形式，相比 Spark 提供 flatMap、join、group 等难以使用；
* 缺乏作业流管理，多步任务需要多次 MapReduce，相比 Spark 提供 DAG 图管理作业流较为复杂。

## 下载安装

在 [Spark 官网](https://spark.apache.org) 下载安装包，这里选择 `2.3.0 (Feb 28 2018)` 和 `Pre-build with user-provided Apache Hadoop`，也就是 `spark-2.3.0-bin-without-hadoop.tgz`。使用之前安装的 Hadoop。

解压

```bash
tar zxvf spark-2.3.0-bin-without-hadoop.tgz 
ln -s spark-2.3.0-bin-without-hadoop spark
cd spark
```

### 配置

使用已经安装的 Hadoop 需要编辑 `conf/spark-env.sh` 文件

```bash
cd conf
cp spark-env.sh.template spark-env.sh
```

因为我将 `hadoop` 命令添加到了 `PATH` 环境变量，这里直接添加下面的内容到 `spark-env.sh` 最后即可

```
export SPARK_DIST_CLASSPATH=$(hadoop classpath)
```

### 运行样例程序

```bash
cd ../
./bin/run-example SparkPi 10
```

这个程序会计算 PI 的近似值，执行之后可以在一团日志中找到

```bash
Pi is roughly 3.1416671416671416
```

### 使用 spark-shell

spark-shell 提供了非常方便的基于 Scala 语言的命令行方式来是哟个 Spark，是学习 Spark 框架的很好方式。

这里以计算 `README.md` 中的 WordCount 为例，首先进入命令行

```bash
./bin/spark-shell
Spark context Web UI available at http://hadoop-host:4040
Spark context available as 'sc' (master = local[*], app id = local-1523582748504).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.0
      /_/
         
Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_161)
Type in expressions to have them evaluated.
Type :help for more information.

scala >
```

可以看到，Spark 的 Web UI 可以从本机的 4040 端口访问。
`Spark context` 与 `Spark session` 已经被初始化为 `sc` 与 `spark` 可以直接在命令行中使用。
当前 Spark 的版本是 2.3.0，Scala 版本是 2.11.8。

```scala
scala> val a = sc.textFile("file:///opt/spark/README.md")
a: org.apache.spark.rdd.RDD[String] = file:///opt/spark/README.md MapPartitionsRDD[1] at textFile at <console>:24

scala> a.count()
res0: Long = 103

scala> 
```

以上两条命令读取了 README.md 并计算了它的行数。

```scala
scala> val w = a.flatMap(l => l.split(" ")).filter(w => w.length() < 20 && w.length() > 0).map(w => (w, 1)).reduceByKey((x,y) => x + y).sortBy(i => i._2, false)
w: org.apache.spark.rdd.RDD[(String, Int)] = MapPartitionsRDD[10] at sortBy at <console>:25

scala> w.take(10)
res0: Array[(String, Int)] = Array((the,24), (to,17), (Spark,16), (for,12), (##,9), (and,9), (a,8), (can,7), (run,7), (on,7))

scala>
```

这里执行了一系列的函数。首先将行通过 `flatMap` 转换为单词，然后通过 `filter` 过滤掉了大于 `20` 的单词和空的单词，再将单词通过 `map` 转换为 key-value, 然后以相同的 key 进行 `reduceByKey`，将 value 相加，最后以相加后的值倒序排序。

最后，通过 `take` 取前十个，也就是出现最多的前十个单词，可以通过结果看到，单词 the 出现了 24 次。

至此，完成了对于 README.md 的单词数目统计，可以看到，依托于 spark 提供的丰富的函数，可以很方便的对数据进行转换。

在输出中可以看到，变量 `a` 的类型是 `org.apache.spark.rdd.RDD[String]`，变量 `w` 的类型是 `org.apache.spark.rdd.RDD[(String, Int)]`。

RDD 也就是 Resilient Distributed Datasets， 在 Spark 2.0 之前，RDD 是主要的 Spark 编程接口，也就是说对于数据的转换操作是围绕着 RDD 来的。
在 Spark 2.0 之后，使用 Dataset 替换了 RDD， Dataset 拥有与 RDD 类似的强类型属性，但在对比 RDD 更加优化。 RDD 依然是支持的，但 Spark 非常推荐开发者转换到 Dataset，因为它有着更好的性能。

使用 Dataset 来编写上面的单词统计程序

```
scala> val a = spark.read.textFile("file:///opt/spark/README.md")
a: org.apache.spark.sql.Dataset[String] = [value: string]

scala> val w = a.flatMap(l => l.split(" ")).filter(w => w.length() < 20 && w.length() > 0).groupByKey(identity).count().sort($"count(1)".desc)
w: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]

scala> w.take(10)
res10: Array[(String, Long)] = Array((the,24), (to,17), (Spark,16), (for,12), (and,9), (##,9), (a,8), (can,7), (on,7), (run,7))

scala> 
```

在运算过程中，可以使用 cache 来缓存结果

```
scale> w.cache()
```

## 使用 spark-sql 统计 Nginx 日志

数据依旧是使用几行 Nginx 日志

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

这里目标是统计各个 IP 的访问次数，简单的使用空格分隔 IP 与其他数据

```scale
scala> case class Log(ip:String, other:String)
defined class Log

scala> val df = sc.textFile("/user/hive/warehouse/tbl_access_log/access_last_10.log").map(_.split(" ")).map(a => new Log(a(0), a(1))).toDF()
df: org.apache.spark.sql.DataFrame = [ip: string, other: string]

scala> df.printSchema()
root
 |-- ip: string (nullable = true)
 |-- other: string (nullable = true)

scala> df.createOrReplaceTempView("log")

scala> spark.sql("SELECT ip, count(*) AS c FROM log GROUP BY ip ORDER BY c DESC").show()
+-------------+---+                                                             
|           ip|  c|
+-------------+---+
|141.8.142.120|  2|
|  5.45.207.60|  2|
|  37.9.113.74|  1|
|  1.199.93.43|  1|
|46.161.55.108|  1|
| 141.8.142.21|  1|
| 141.8.183.10|  1|
|  5.45.207.41|  1|
+-------------+---+


scala> 
```

这里首先定义了一个类 `Log` 用来表达一条日志，然后将 RDD 通过 `toDF` 转换为 DataFrame，然后 `df.createOrReplaceTempView` 创建了一个 `View` 类似于关系型数据库中的视图概念。
最后通过执行一条 SQL 语句，很方便的查询出访问最多的几个 IP 地址。

DataFrame 是组织到命名列中的 Dataset， 概念上等同于关系型数据库中的表。
DateFrame 可以从各种来源构建，比如结构化的数据文件、Hive Table、外部数据库，现有的 RDD，在 Scale API 中 DataFrame 只是 Dataset[Row] 的别名，在 Java API 中，开发者需要使用 Dataset<Row> 来表示 DataFrame。

Dataset 编程接口从 Spark 1.6 添加进来。

## 参考链接

* [Spark Quick Start](https://spark.apache.org/docs/latest/quick-start.html)
* [Spark RDD Programming Guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html)
* [Spark SQL, DataFrames and Datasets Guide](https://spark.apache.org/docs/latest/sql-programming-guide.html)






