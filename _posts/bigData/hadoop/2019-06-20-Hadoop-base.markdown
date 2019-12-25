---
layout:     post
title:      "Hadoop基础"
subtitle:   " \"The Apache™ Hadoop® project develops open-source software for reliable, scalable, distributed computing \""
date:       2019-06-20 02:30:00
author:     "YaPi"
header-img: "img/bigData/hadoop-logo.jpg"
tags:
    - 大数据
---


### Hadoop初识

- 狭义的Hadoop: 是一个舍和大数据分布式存储(HDFS)、分布式计算(MapReduce)和资源调度(YARN)的平台

- 广义Hadoop: 指的是Hadoop的生态系统,包含各种只解决某一个特定问题域的子系统，zk、hiv(sql查询)、pig(脚本)、R语言、Hbase、sqoop(将关系型数据库与hadoop的数据进行交换)、Flume(日志收集框架)


#### 基础

- 定义
    1. 提供分布式的存储(一个文件被拆分成很多歌快,并且以副本的方式存储在各个节点中)和计算
    2. 是一个分布式的系统基础架构: 用户可以在不了解分布式底层细节的情况下进行使用
- 核心模块
1. Hadoop Common
2. Hadoop Distributed File System 【(HDFS) 分布式文件系统,将文件分布式存储在很多的服务器上】
3. Hadoop YARN(作业调度资源管理)
4. Haddop MapReduce(分布式计算框架,在很多机器上分布式并行计算)


#### HDFS

- 源自Google的GFS论文
- HDFS是GFS的克隆版
- HDFS的特点(扩展性、容错性、海量数据存储)

##### 特性

1. 将文件切分成指定大小的数据快,并以多副本的方式存储在多个机器上
    1. 块(block): 默认的blocksize是128M
    2. 副本: 默认3个副本
2. 数据切分,多副本容错,操作对用户透明

#### MapReduce

- 源自Google的MapReduce论文
- MapReduce是Google MapReduce的克隆版
- MapReduce特点: 扩展性、容错性、海量数据离线处理


#### YARN

- YARN : Yet Another Resource Negotiiator
- 负责整个集群资源的管理和调度
- YARN特点: 扩展性、容错性、多框架资源统一调度

#### 优势

- 高可靠性
    1. 数据存储: 数据多副本
    2. 运行可靠
- 搞扩展性
    1. 存储/计算资源不够时,可以横向的线性扩展机器
    2. 一个集群中可以包含数以千计的节点
- 成熟的生态圈


#### 发行版本选择
- Apache
    1. 纯开源
    2. 不同版本、不同框架整合不方便

- CDH: https://www.cloudera.com
    1. cm (cloudera manager) 通过页面一键安装各种框架,方便升级,支持impala
    2. cm不开源,与社区版本有些出入
- Hortonworks: HDP
    1. 纯开源,支持tez,企业发布自己的数据平台可以直接基于页面框架改造
    2. 企业级安全不开源
- MapR

#### 使用CDH安装

- CDH 相关软件包下载地址
    > http://archive.cloudera.com/cdh5/cdh/5/
- 使用版本：hadoop-2.6.0-cdh5.15.1
- Hadoop 下载
- JDK 下载
- hive 下载

#### 安装

##### JDK 安装

1. 拷贝JDK到虚拟机
```
scp jdk-8u201-linux-x64.tar.gz  hadoop@172.16.114.132:~/software/
```
2. 解压到～/app/ 目录
```
tar -zxvf jdk-8u201-linux-x64.tar.gz  -C ~/app/
```
3. 配置无密码登陆
    1. ssh-keygen -t rsa
    2. cat id_rsa.pub >> authorized_keys
    3. chmod 600 authorized_keys

##### Hadoop安装
- 下载安装包拷贝到虚拟机
- 解压到对应目录
- 目录结构
    1. bin : hadoop客户端命令
    2. etc/hadoop : 相关配置文件目录
    3. sbin : 启动hadoop相关进程的脚本
    4. lib : 依赖包
    5. share : 常用例子


##### 基础配置

- /etc/hadoop/hadoop-env.sh （配置JAVA_HOME）
- /etc/hadoop/core-site.xml  （配置文件系统主节点)

```
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop000:8020</value>
</property>
```
- /etc/hadoop/hdfs-site.xml (配置文件副本系数)

```
<property>
    <name>dfs.replication</name>
    <value>1</value>
</property>

默认文件存储目录在 /tmp/目录下,这个目录在重启的时候会清空
所以重新指定一下目录
<property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/app/tmp</value>
</property>
```

- slaves (单机的时候修改 localhost为hadoop000(自己设置的))

##### 启动

- 第一次执行的时候一定要格式化文件系统,不要重复执行(hdfs namenode -format)
- 启动集群 /sbin/start-dfs.sh
- 查看是否启动成功,输入命令 jps   --> 显示应该有4个进程
- 启动日志 /home/hadoop/app/hadoop-2.6.0-cdh5.15.1/logs/
- 启动成功过后默认hdfs端口为 50070,可浏览器访问,若访问不到,可能是防火墙原因
    1. 查看防火墙是否打开 firewall -cmd --state
    2. 若打开,可以将其关闭 sudo systemctl stop firewalld.service
- start/stop-dfs.sh 与 hadoop-daemons.sh的关系
    1. start/stop-dfs.sh 是后面那个命令的组合
    2. hadoop-daemons.sh start namenode
    3. hadoop-daemons.sh start datanode



