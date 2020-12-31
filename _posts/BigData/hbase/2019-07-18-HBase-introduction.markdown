---
layout:     post
title:      "HBase入门及简单应用"
subtitle:   " \"Apache HBase is the Hadoop database, a distributed, scalable, big data store \""
date:       2019-07-18 02:30:00
author:     "YaPi"
header-img: "img/bigData/hbase_logo_with_orca_large.png"
tags:
    - 大数据
---


## HBase

1. [HBase简介](#hBase简介)
2. [HBase安装](#hbase安装)
3. [HBase读写流程](#hbase读写流程)
4. [HBase实战使用](#hbase实战使用)

#### HBase简介

> 面向列、适合非结构化数据存储

HBase是一个分布式的、面向列的开源数据库，该技术来源于 Fay Chang 所撰写的Google论文“Bigtable：一个结构化数据的分布式存储系统”。就像Bigtable利用了Google文件系统（File System）所提供的分布式数据存储一样，HBase在Hadoop之上提供了类似于Bigtable的能力。HBase是Apache的Hadoop项目的子项目。HBase不同于一般的关系数据库，它是一个适合于非结构化数据存储的数据库。另一个不同的是HBase基于列的而不是基于行的模式

##### 在生态圈所处地位:

- HBase是Apache基金会顶级项目
- HBase基于Hadoop的核心HDFS系统进行数据存储
- HBase可以存储超大数据并使用用来进行大数据的==实时查询==

##### HBase & HDFS

- HBase建立在Hadoop文件系统之上,利用了Hadoop文件系统的容错能力
- HBase提供对数据的随机实时读写/访问的能力
- HBase内部使用哈希表，并存储索引，可将HDFS文件中数据进行快速查找

##### HBase & 常用关系型数据库

> 事物支持 只支持单个Row级别，索引只支持Row-key

- NameSpace: 可以把NameSpace理解为RDBMS的数据库
- Table: 表名必须是能用在文件路径里的合法名字
- Row: 在表里面，每一行代表着一个数据对象，每一行都是以一个行键(Row Key)来进行唯一标识的，行键并没有什么特定的数据类型，以二进制的字节来存储
- Column: HBase的列由Column family和Column qualifier组成，由冒号(:)进行间隔。比如famiy:qualifier
- RowKey: 可以唯一标识一行记录，不可被改变
- Column Family: 在定义HBase表的时候需要提前设置好列族，表中所有列都需要组织在列族里面
- Column Qualifier: 列族中的数据通过列标识来进行映射，可以理解为一个键值对，Column Qualifier就是key
- Cell: 每一个行键，列族和列标识共同组成一个单元
- Timestamp: 每一个值都会有一个timestamp，作为该值特定版本的标识符，由HBase管理,默认保存3个保本

关系型数据库存储方式:

ID | File Name | File Size | creator
---|--- |---|---
1 | a.txt | 20 | 张三
2 | b.txt | 30 | 李四

HBase存储:

将File Name和File Size抽象成一个列族,将creator抽象成一个列族

RowKey | FileInfo | SaveInfo
---|--- |---|---
1 | name:a.txt size:20 | creator:张三
2 | name:b.txt size:30 | creator:李四



##### 使用场景

- 瞬间写入量很大,常用数据不好支撑或需要很高成本
- 数据需要长久保存，且量会持久增长到比较大的场景
- HBase不适用于有join，多级索引，表关系复制的数据模型


### HBase安装

1. 下载对应安装包,找到对应Hadoop版本下载,并解压
2. 复制 Hadoop内 core.site.xml和hdfs-site.xml文件到HBase的conf目录内
3. 修改hbase-env.sh
    1. export JAVA_HOME=/home/hadoop/app/jdk8
    2. 找到  Configure PermSize. Only needed in JDK7 ... 注释掉下面的两行(HBASE_MASTER_OPTS 、HBASE_REGIONSERVER_OPTS)
4. 修改 hbase-site.xml

```
<configuration>
    <property>
        <name>hbase.rootdir</name>
        # 和hadoop启动的端口一致
        <value>hdfs://hadoop000:8020/hbase</value>
    </property>
    <property>
        # 新建存储目录
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/home/hadoop/app/hbasetmp</value>
    </property>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
</configuration>
```

### HBASE读写流程

#### HBASE 写流程

- Client会先访问zookeeper,得到对应的RegionServer地址
- Client对RegionServer发起写请求，RegionServer接受数据写入内存
- 当MemStrore的大小达到一定的值后,flush到StoreFile并存储到HDFS


名次释意:

1. RegionServer : 管理多个 Region, 每一个RegionServer对应一个HLog
2. Region : 包含多个Store,每一个Store 包含 MemStore 和 StoreFile，写入数据的时候,先写入到MemStore中，当MemStore满了,
再写入到 StoreFile
3. StoreFile : 对应HDFS中的HFile,是其一层封装
4. WAL : Write-Ahead Logging, 预写日志系统
数据库中一种高效的日志算法，对于非内存数据库而言，磁盘I/O操作是数据库效率的一大瓶颈。在相同的数据量下，采用WAL日志的数据库系统在事务提交时，磁盘写操作只有传统的回滚日志的一半左右，大大提高了数据库磁盘I/O操作的效率，从而提高了数据库的性能。
一般的WAL的实现方式是先写入日志,再写入内存,HBase是先写入内存,再写到HLog中(更新操作)
5. HLog : WAL的一种实现方式,只有当HLog更新了,才表示此条数据写入成功。


#### HBASE 读流程

- Client会先访问zookeeper,得到对应的RegionServer地址,拉去meta信息表
- Client对RegionServer发起读请求，RegionServer根据meta信息开始读取数据
- 扫描memStore --> 扫描blockcache -- > 从storefile中读取数据


#### 三个问题

1. HBase启动时发生了什么
    1. HMaster启动,注册Zookeeper,等待RegionServer汇报
    2. RegionServer注册到Zokeeper,并向HMaster汇报
    3. 对各个RegionServer(包括失效的)的数据进行数据整理,分配Region和meta信息
    4. 其他backupMaster定期向activeMaster进行更新，保证meta的最新
2. 当RegionServer失效后会发生什么
    1. HMaster将失效的RegionServer上的Region分配到其他节点
    2. HMaster更新 hbase:meta 表以保证数据正常访问
3. 当HMaster失效后会发生什么
    1. 处于Backup状态的HMaster节点推选出一个转为Active状态
    2. (若没有配置高可用,activemaster失效后)数据能正常读写,但是不能创建删除表,也不能更改表结构


### HBase实战使用

#### SHELL操作命令


名称 | 命令表达式
---|---
查看存在哪些表 | list
创建表 | create '表名称','列名称1','列名称2','列名称N'
添加记录 | put '表名称','行名称','列名称','值'
查看记录 | get '表名称','行名称'
查看表中的记录总数 | count '表名称'
删除记录 | delete '表名','行名称','列名称'
删除一张表 | 先要屏蔽该表,才能对该表进行删除,第一步 disable '表名称' 第二步 drop '表名称'
查看所有记录 | scan '表名称'
查看某个表某个列中的所有数据 | scan '表名称',['列名称:']
更新记录 | 就是重写一边进行覆盖


```
进入Hbase命令行操作 ： $HBASE_HOME/bin/hbase.sh shell

查看集群状态 ： status

查看所有的表 : list

创建一张表并制定列族 : create 'FileTable','saveInfo','tableInfo'

添加一个列族 : alter 'FileTable','cf'

删除一个列族 : alter 'Filetable',{NAME=>'cf',METHOD=>'delete'}

添加数据 : put 'FileTable','rowkey1','fileInfo:size','10241'

查看表数据 : get 'FileTable','rowkey1'
    查看指定列族 : get 'FileTable','rowkey1','fileInfo'

查看所有数据 : scan 'FileTable'
    查看指定列 : scan 'FileTable',{COLUMN=>'fileinfo'}
    查看 从 rowkey1开始，返回2条数据,版本是最行的
    scan 'FileTable',{STARTROW=>'rokey1',LIMIT=>2,VERSIONS=>1}

删除列中某一个值 : delete 'FileTable','rowkey1','fileInfo:size'
删除整个列 : delteall 'FileTable','rowkey1'

删除整个表 :
    1. disable 'FileTable'
    2. drop 'FileTable'

查看表状态 is_dsiabled 'FileTable'
查看是否存在 exists 'FileTable'
```

#### HBase JAVA API

##### 开发HBase数据库操作类

###### 过滤器

HBase为筛选数据提供了一组过滤器,通过过滤器可以在数据的
多个纬度上进行数据的筛选(行、列、数据版本),通常来说对行
列进行筛选的应用场景较多

1. 基于行的过滤器
    1. PrefixFilter: 行的前缀匹配
    2. PageFilter: 基于行的分页
2. 基于列的过滤器
    1.  ColumnPrefixFilter: 列前缀匹配
    2.  FirstKeyOnlyFilter: 只返回每一行的第一列
3. 基于单元值的过滤器
    1. KeyOnlyFilter: 返回的数据不包括单元值,只包含行键与列
    2. TimestampsFilter: 根据数据的时间戳版本进行过滤
4. 基于列和单元值的过滤器
    1. SingleColumnValueFilter : 对该列的单元值进行比较过滤
    2. SingleColumnValueExcludeFilter: 对该列的单元值进行比较过滤
5. 比较过滤器
    1. 比较过滤器通常需要一个比较运算符以及一个比较器来实现过滤
    2. RowFilter、FamilyFilter、QualifierFilter、ValueFilter

6. 自定义过滤器
    1. 实现FilterBase类 实现相应方法


##### 简单使用

###### 引入包

```
<dependencies>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.2.4</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
</dependencies>

```

###### 获取数据库连接

```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Table;

import java.io.IOException;

public class HBaseConn {

    private static final HBaseConn INSTANCE = new HBaseConn();

    private static Configuration configuration;

    private static Connection connection;

    private HBaseConn(){
        try {
            if (configuration == null){
                configuration = HBaseConfiguration.create();
                // zookeeper 地址及端口
                configuration.set("hbase.zookeeper.quorum","172.16.54.134:2181");
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    // 获取连接
    private Connection getConnection(){
        if (connection == null || connection.isClosed()){
            try {
                // 设置连接配置
                connection = ConnectionFactory.createConnection(configuration);
            }catch (Exception e){
                e.printStackTrace();
            }
        }

        return connection;
    }

    // 获取连接
    public static  Connection getHBaseConn(){
        return INSTANCE.getConnection();
    }

    // 获取表
    public static Table getTable(String tableName) throws IOException {
        return INSTANCE.getConnection().getTable(TableName.valueOf(tableName));
    }

    // 关闭连接
    public static void closeConn(){
        if (connection != null){
            try {
                connection.close();
            }catch (IOException e){
                e.printStackTrace();
            }
        }
    }
}

```

###### 创建相应方法

```
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.filter.FilterList;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

public class HBaseUtil {

    /**
     * @description:
     * @date: 2019/07/17
     * @param: [tableName, cfs]
     * 表名 列族的数组
     * @return: boolean
     */
    public static boolean createTable(String tableName,String[] cfs){

        try (HBaseAdmin admin= (HBaseAdmin)HBaseConn.getHBaseConn().getAdmin()) {
            if (admin.tableExists(tableName)){
                return false;
            }
            HTableDescriptor tableDescriptor = new HTableDescriptor(TableName.valueOf(tableName));

            Arrays.stream(cfs).forEach( cf -> {
                HColumnDescriptor columnDescriptor = new HColumnDescriptor(cf);
                columnDescriptor.setMaxVersions(1);
                tableDescriptor.addFamily(columnDescriptor);
            });

            admin.createTable(tableDescriptor);
        }catch (Exception e){
            e.printStackTrace();
        }
        return true;
    }

    public static boolean deleteTable(String tablename){
        try (HBaseAdmin admin= (HBaseAdmin)HBaseConn.getHBaseConn().getAdmin()){
            admin.disableTable(tablename);
            admin.deleteTable(tablename);
        }catch (Exception e){
            e.printStackTrace();
        }
        return true;
    }

    /*
     * @description:
     * @date: 2019/07/17
     * @param: [tableName, rowkye, cfName, qualifire, data]
     * 表名 唯一标识 列族名 列标识 数据
     * @return: boolean
     */
    public static boolean putRow(String tableName,
                                String rowKey,
                                String cfName,
                                String qualifier,
                                String data){
        try (Table table = HBaseConn.getTable(tableName)){

            Put put = new Put(Bytes.toBytes(rowKey));
            put.addColumn(Bytes.toBytes(cfName),
                    Bytes.toBytes(qualifier),
                    Bytes.toBytes(data));

            table.put(put);
        }catch (IOException e){
            e.printStackTrace();
        }

        return true;
    }

    // 批量插入
    public static boolean putRows(String tableName, List<Put> rows){

        try (Table table = HBaseConn.getTable(tableName)){

            table.put(rows);
        }catch (IOException e){
            e.printStackTrace();
        }
        return true;
    }

    // 读取一行数据
    public static Result getRow(String tableName, String rowKey){
        try (Table table = HBaseConn.getTable(tableName)){

            Get get = new Get(Bytes.toBytes(rowKey));
            return table.get(get);

        }catch (IOException e){
            e.printStackTrace();
        }

        return null;
    }


    // 通过过滤器获取数据
    public static Result getRow(String tableName,
                                String rowKey,
                                FilterList filterList){
        try (Table table = HBaseConn.getTable(tableName)){

            Get get = new Get(Bytes.toBytes(rowKey));

            get.setFilter(filterList);
            return table.get(get);

        }catch (IOException e){
            e.printStackTrace();
        }

        return null;
    }

    // 使用scan扫描
    public static ResultScanner getScanner(String tableName){
        try (Table table = HBaseConn.getTable(tableName)){

           Scan scan = new Scan();
           scan.setCaching(1000);
           return table.getScanner(scan);

        }catch (IOException e){
            e.printStackTrace();
        }

        return null;
    }


    // 全表扫描 指定开始结束 rowKey
    // 只包含rowkey1 不包含rowkey2
    public static ResultScanner getScanner(String tableName,
                                           String startRowKey,
                                           String endRowKey){
        try (Table table = HBaseConn.getTable(tableName)){

            Scan scan = new Scan();

            scan.setStartRow(Bytes.toBytes(startRowKey));

            scan.setStopRow(Bytes.toBytes(endRowKey));

            scan.setCaching(1000);

            return table.getScanner(scan);

        }catch (IOException e){
            e.printStackTrace();
        }

        return null;
    }

    // 全表扫描 指定开始结束 rowKey
    // 指定过滤器
    public static ResultScanner getScanner(String tableName,
                                           String startRowKey,
                                           String endRowKey,
                                           FilterList filterList){
        try (Table table = HBaseConn.getTable(tableName)){

            Scan scan = new Scan();

            scan.setFilter(filterList);

            scan.setStartRow(Bytes.toBytes(startRowKey));

            scan.setStopRow(Bytes.toBytes(endRowKey));

            scan.setCaching(1000);

            return table.getScanner(scan);

        }catch (IOException e){
            e.printStackTrace();
        }

        return null;
    }

    // 删除某一行
    public static boolean deleteRow(String tableName, String rowKey){

        try (Table table = HBaseConn.getTable(tableName)){

            Delete delete = new Delete(Bytes.toBytes(rowKey));

            table.delete(delete);

        }catch (IOException e){
            e.printStackTrace();
        }

        return true;
    }

    // 删除列族
    public static boolean deleteCloumn(String tableName, String cfName){

        try (HBaseAdmin admin= (HBaseAdmin)HBaseConn.getHBaseConn().getAdmin()) {
            admin.deleteColumn(tableName,cfName);
        }catch (Exception e){
            e.printStackTrace();
        }
        return true;
    }

    // 删除列族中的qualifier
    public static boolean deleteQulifier(String tableName,
                                         String rowKey,
                                         String cfName,
                                         String qualifierName){
        try (Table table = HBaseConn.getTable(tableName)){

            Delete delete = new Delete(Bytes.toBytes(rowKey));

            delete.addColumn(Bytes.toBytes(cfName),
                    Bytes.toBytes(qualifierName));

            table.delete(delete);

            table.delete(delete);

        }catch (IOException e){
            e.printStackTrace();
        }

        return true;
    }


}

```

###### 测试

```
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.client.ResultScanner;
import org.apache.hadoop.hbase.filter.*;
import org.apache.hadoop.hbase.util.Bytes;
import org.junit.Test;

import java.util.Arrays;

public class HBaseUtilTest {

    @Test
    public void createTable(){
        HBaseUtil.
                createTable("FileTable",new String[]{"fileInfo","saveInfo"});
    }

    @Test
    public void addFileDetails(){
        HBaseUtil.putRow("FileTable","rowkey3",
                "fileInfo","name","file11.txt");
        HBaseUtil.putRow("FileTable","rowkey3",
                "fileInfo","type","txt");
        HBaseUtil.putRow("FileTable","rowkey3",
                "fileInfo","size","1024");
        HBaseUtil.putRow("FileTable","rowkey3",
                "fileInfo","creator","ergou");


        HBaseUtil.putRow("FileTable","rowkey2",
                "fileInfo","name","file22.txt");
        HBaseUtil.putRow("FileTable","rowkey2",
                "fileInfo","type","txt");
        HBaseUtil.putRow("FileTable","rowkey2",
                "fileInfo","size","2048");
        HBaseUtil.putRow("FileTable","rowkey2",
                "fileInfo","creator","dagou");
    }


    @Test
    public void getFileDetail(){
        Result result = HBaseUtil.getRow("FileTable",
                "rowkey2");

        if (result != null){
            System.out.println("rowKey = " + Bytes.toString(result.getRow()));

            System.out.println("fileName = " + Bytes.toString(
                    result.getValue(Bytes.toBytes("fileInfo"),
                            Bytes.toBytes("name"))
            ));
        }
    }

}
```