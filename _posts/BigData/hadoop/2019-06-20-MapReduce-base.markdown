---
layout:     post
title:      "MapReduce基础"
subtitle:   " \"Hadoop MapReduce is a software framework for easily writing applications which process vast amounts of data (multi-terabyte data-sets) in-parallel on large clusters (thousands of nodes) of commodity hardware in a reliable, fault-tolerant manner. \""
date:       2019-06-20 02:30:00
author:     "YaPi"
header-img: "img/bigData/hadoop-logo.jpg"
tags:
    - 大数据
---

#### 概述

- 源自与谷歌的MapReduce的论文,发表于2004年12月
- Hadoop MapReduce 是 Google MapReduce的克隆版
- MapReduce的有点 : 海量数据的离线处理&易开发&易运行
- MapReduce的缺点 : 实时流式计算

```
input --> spliting --> Maping -->

shuffling(将不同的key分到一起) --> Reduceing --> Final result
```


##### 读取流程

- InputFormat 读取Hdfs里面的数据
- 读取的数据转为Split
- Split转为RecordReader(一行数据)
- Map
- Partition（Shuffle）过程
- Reduce


##### MapReduce编程模型核心概念
- Split
    1.  将输入的数据进行拆分,拆分为RecordReader(一行数据)
- InputFormat
    1. 将RecordReader进行读取,处理
- OutPutFormat
    1. 将处理了的数据写出到指定地方
- Combiner
    1. 设置Combiner: job.setComnierClass()
    2. 在map端做一个聚合,后再将数据传到Reduce,这个操作其实和Reduce的逻辑是一样的
    3. 算除法的时候,比如平均数的时候,先聚合的话会导致分母出问题,需要慎重考虑
- Partitioner  -->按key分发map后的值

#### 使用自定义类型进行MapReduce


##### 自定义Hadoop类型处理类
- 实现Writable接口
- 定义空构造函数
- 复写write、readFields方法
    1. write: 将各个字段写入进去 out.writeUTF(phone) ....
    2. readFields: 将字段的值读取出来 in.readUTF()....
    3. 读取字段的值的顺序必须和写入字段的顺序相同

##### 自定义Mapper

1. 定义使用
2. 复写map方法,分割数据,进行整理

```
public class AccessMapper extends
        Mapper<LongWritable, Text,Text,Access> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String[] lines =
                value.toString().split("\\s+");

        String phone = lines[1];// 取出手机号
        long up = Long.parseLong(lines[lines.length-3]);// 取出上行流量
        long down = Long.parseLong(lines[lines.length-2]);// 取出下行流量

        context.write(
                new Text(phone),
                new Access(phone,up,down));
    }
}
```

##### 自定义Reducer

1. 继承Reducer
2. 复写Reduce方法

```
public class AccessReducer extends
        Reducer<Text,Access,Text,Access> {

    @Override
    protected void reduce(Text key, Iterable<Access> values, Context context) throws IOException, InterruptedException {
        long ups = 0;
        long downs = 0;

        for(Access access : values){
            ups += access.getUp();
            downs += access.getDown();
        }

        context.write(key,
                new Access(key.toString(),ups,downs));
    }
}

//若不想再输出key,只想输出Access的信息

public class AccessReducer extends
        Reducer<Text,Access, NullWritable,Access> {
            ...

            context.write(
                NullWritable.get(),
                new Access(key.toString(),ups,downs));
        }

```

##### 默认shuffle规则

HashPartitioner 是默认的分区方式
规则 : 分发的key的hash值与reduce task的个数取模

```
public class HashPartitioner<K, V> extends Partitioner<K, V> {
    public HashPartitioner() {
    }
    // numReduceTasks : 指定的Reducer的个数,决定了reduce作业输出文件的个数
    public int getPartition(K key, V value, int numReduceTasks) {
        return (key.hashCode() & 2147483647) % numReduceTasks;
    }
}
```

##### 自定义分区规则

1. 继承Partitioner两个范型为map输出的key和value值
2. 复写getPartition方法
3. Job中设置Partition和分区数量

```
public class AccessPartitioner extends
        Partitioner<Text,Access> {

    @Override
    public int getPartition(Text phone, Access access, int i) {
        // 将15开头的数据,放在第0个
        if(phone.toString().startsWith("15")){
            return 0;
        }else if (phone.toString().startsWith("17")){
            return 1;
        }else{
            return 2;
        }
    }
}

//JOB
// 设置自定义分区规则
job.setPartitionerClass(AccessPartitioner.class);
// 设置分区个数
job.setNumReduceTasks(3);


```


##### 启动

```
public static void main(String args[])throws Exception{
        Configuration configuration = new Configuration();

        // 设置参数 可在此指定调用远程hdfs文件系统
        Job job = Job.getInstance(configuration);

        // 设置启动类名称
        job.setJarByClass(AccessApp.class);
        // 设置自定义map类
        job.setMapperClass(AccessMapper.class);
        // 设置自定义reduce类
        job.setReducerClass(AccessReducer.class);

        // 设置map输出的key,输入的key默认为LongWritable(表示行号)
        job.setMapOutputKeyClass(Text.class);
        // 设置map输出的value, 输入的value默认为那一行数据
        job.setMapOutputValueClass(Access.class);

        // 设置reduce输出的key的类型
        job.setOutputKeyClass(Text.class);
        // 设置reduce输出的value的类型
        job.setOutputValueClass(Access.class);


        // 设置自定义分区规则
        job.setPartitionerClass(AccessPartitioner.class);
        // 设置分区个数
        job.setNumReduceTasks(3);

        // 设置输入文件路径
        FileInputFormat.setInputPaths(
                job,
                new Path("access/input")
        );

        // 输出文件路径
        FileOutputFormat.setOutputPath(
                job,
                new Path("access/output")
        );
        job.waitForCompletion(true);

    }
```