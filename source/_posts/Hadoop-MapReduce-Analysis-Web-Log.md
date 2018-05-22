title: "使用 Hadoop MapReduce 分析日志"
date: 2018-05-20 17:20:17
tags:
- Hadoop
- 大数据
- MapReduce
categories: 
- 大数据

---



## 概述

本例使用 MapReduce 对 Web 日志中客户端 IP 进行统计

Mapper 的输入是 Web 日志，如下

```
10.168.1.73 ...
10.1.1.2 ...
10.1.1.2 ...
```

Mapper 程序以数据的偏移为 Key，日志行为 Value 输入。数据偏移对于本例可以忽略，对输入行以空格分隔后第一个元素即为 IP。

Mapper 的输出是以 IP 为 Key，由于一行只有一个 IP，直接以 1 作为 Value。如下

```
(10.168.1.73, 1)
(10.1.1.2, 1)
(10.1.1.2, 1)
```

经过 Shuffle （混洗），数据被处理为

```
(10.168.1.73, [1])
(10.1.1.2, [1, 1])
```

混洗的过程可以描述为

Mapper 对相同 Key 的数据进行聚合，然后复制到 Reducer 节点，Reducer 节点对来自多个 Mapper 节点的相同 Key 数据再进一步聚合。这里保证了相同 Key 的数据会被放到一个 Reducer 中。

Reducer 接收到数据，对每个 Key 的 Value 相加，输出结果

```
(10.168.1.73, 1)
(10.1.1.2, 2)
```

至此，整个 MapReduce 完成。

处理大数据量数据时，为减少网路 IO， Mapper 程序会在靠近数据的节点上运行，优先在数据节点上运行，若数据节点上存在繁忙的 Mapper 程序，则会在同一机架的节点上运行，最后才会挑选不同机架的节点上运行。

Mapper 处理完成时需要将输出传输到 Reducer 节点，这部分需要进行网络传输。为减少这部分的网络 IO，引入了 Combiner 函数。
Combiner 函数就像运行在 Mapper 节点上的 Reducer 一样，将一个 Mapper 的输出先进行处理。
在本例中，Combiner 函数可以将 Mapper 输出的次数先进行相加。也即：

Mapper 1 输出为

```
(10.168.1.73, 1)
(10.1.1.2, 1)
```

Mapper 2 输出为

```
(10.1.1.2, 1)
(10.1.1.2, 1)
```

在不使用 Combiner 时，直接将 Mapper 输出传输到 Reducer 节点。若使用 Combiner 函数，

Mapper 1 Combiner 输出为

```
(10.168.1.73, 1)
(10.1.1.2, 1)
```

Mapper 2 Combiner 输出为

```
(10.1.1.2, 2)
```

可以减少传输到 Reducer 的数据量。

Combiner 函数并不是在任何时刻都可以使用，例如在计算平均值时，各个 Mapper 的平均值在 Reducer 上再求平均值与整体求平均值很可能是不同的。

## 代码

样例代码为

```java
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class LogMain {

    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        Job job = Job.getInstance();
        job.setJobName("Log Analysis: Request Times per IP");
        job.setJarByClass(LogMain.class);

        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));

        job.setMapperClass(LogMapper.class);
        /*
         指定 Combiner，Combiner 同样是继承 Reducer，
         此处 Combiner 的逻辑与 Reducer 的逻辑相同，指定相同的类
          */
        job.setCombinerClass(LogReducer.class);
        job.setReducerClass(LogReducer.class);

        // 此处 Reducer 输出与 Mapper 输出类型相同，无需对 Mapper 指定输出类型，否则需要为 Mapper 指定输出类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(LongWritable.class);

        System.exit(job.waitForCompletion(true) ? 0: 1);
    }

}

/**
 * 此处指定泛型依次是 Mapper 的输入 Key 类型，Mapper 的输入 Value 类型，Mapper 的输出 Key 类型，Mapper 的输出 Value 类型
 */
class LogMapper extends Mapper<LongWritable, Text, Text, LongWritable> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String line = value.toString();
        String[] split = line.split(" ", 2);
        String ip = "";
        long count = 0;
        if (split.length > 1) {
            ip = split[0];
            count = 1;
        }
        context.write(new Text(ip), new LongWritable(count));
    }
}

/**
 * 此处指定的泛型依次是 Reducer 的输入 Key 类型，Reducer 的输入 Value 类型，Reducer 的输出 Key 类型，Reducer 的输出 Value 类型
 */
class LogReducer extends Reducer<Text, LongWritable, Text, LongWritable> {

    @Override
    protected void reduce(Text key, Iterable<LongWritable> values, Context context) throws IOException, InterruptedException {
        long count = 0;
        for (LongWritable l : values) {
            count += l.get();
        }
        context.write(key, new LongWritable(count));
    }
}
```

输入数据

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

分别执行命令

```
# 编译
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
hadoop com.sun.tools.javac.Main LogMain.java
# 打包
jar cf demo.jar *.class
# 准备数据
hadoop fs -mkdir -p /user/hadoop/log_analysis/input
hadoop fs -put access_last_10.log /user/hadoop/log_analysis/input/
# 执行
hadoop jar demo.jar LogMain /user/hadoop/log_analysis/input /user/hadoop/log_analysis/output
# 查看结果
hadoop fs -ls /user/hadoop/log_analysis/output
hadoop fs -cat /user/hadoop/log_analysis/output/part-r-00000
```

可以看到结果

```
1.199.93.43     1
141.8.142.120   2
141.8.142.21    1
141.8.183.10    1
37.9.113.74     1
46.161.55.108   1
5.45.207.41     1
5.45.207.60     2
```


## 参考链接

《Hadoop 权威指南》
