title: "第一个 MapReduce 程序"
date: 2018-04-11 13:20:17
tags:
- Hadoop
- 大数据
- MapReduce
categories: 
- 大数据

---

## 概述

Hadoop MapReduce 可以方便的编写运行与数千台计算节点集群的大数据量处理程序。

MapReduce 是以 Key-Value 方式处理数据的，标准流程:

```
(input) <k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)
```

## 第一个 MapReduce 程序

第一个 MapReduce 程序是一个简单的单词统计，即 WordCount：

Maven 中添加依赖

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>3.1.0</version>
</dependency>

```

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class MapReduceDemo01 {

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Job job = Job.getInstance(new Configuration());
        job.setJarByClass(MapReduceDemo01.class);

        job.setMapperClass(Mapper.class);
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(LongWritable.class);

        job.setReducerClass(Reducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);

        FileInputFormat.addInputPath(job, new Path("/user/hadoop/word_count/input"));
        FileOutputFormat.setOutputPath(job, new Path("/user/hadoop/word_count/output"));
        job.waitForCompletion(true);
    }
}

class Mapper extends org.apache.hadoop.mapreduce.Mapper<LongWritable, Text, Text, LongWritable> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] words = line.split(" ");
        for (String w : words) {
            context.write(new Text(w), new LongWritable(1));
        }
    }
}

class Reducer extends org.apache.hadoop.mapreduce.Reducer<Text, LongWritable, Text, LongWritable> {

    @Override
    protected void reduce(Text key, Iterable<LongWritable> values, Context context) throws IOException, InterruptedException {
        long count = 0;
        for (LongWritable i : values) {
            count += i.get();
        }
        context.write(key, new LongWritable(count));
    }
}
```

代码较为简单，只有一个 java 文件，设置了 Map 类与 Reduce 类，并分别设置了他们的输出 Key-Value 的类型。最后设置了任务的输入输出目录。

## 运行

### 配置环境变量

```bash
export JAVA_HOME=/usr/lib/jvm/java-openjdk
export PATH=${JAVA_HOME}/bin:${PATH}
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
```

建议将环境变量写入 `/etc/profile` 

### 编译

```bash
./bin/hadoop com.sun.tools.javac.Main MapReduceDemo01.java
```

### 打 jar 包

```bash
jar cf demo.jar *.class
```

### 准备输入数据

```bash
./bin/hadoop fs -appendToFile - /user/hadoop/word_count/input/input-01.txt
```

### 执行

```bash
./bin/hadoop jar demo.jar MapReduceDemo01
```

### 查看结果

执行结果被保存在我们指定的输入目录中

```bash
./bin/hadoop fs -ls /user/hadoop/word_count/output
```

可以看到有 `_SUCCESS` 与 `part-r-00000` 两个文件，查看 `part-r-00000` 的内容

```bash
./bin/hadoop fs -cat /user/hadoop/word_count/output/part-r-00000
```

至此第一个 MapReduce 程序执行完毕

## 总结

这里使用 Java 编写了第一个 MapReduce 程序。MapReduce 程序也可以使用其他语言进行编写：

* Hadoop streaming 可以使用任何的可执行文件作为 Mapper 和 Reducer，例如 shell，C/C++ 程序，Python程序。
* Hadoop Pipes 使用 SWIG 让 C/C++ 可以编写 MapReduce 程序。

## 参考链接

[MapReduce Tutorial](http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)


