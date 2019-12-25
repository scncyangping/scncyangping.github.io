---
layout:     post
title:      "HBase优化、协处理器、容灾"
subtitle:   " \"Apache HBase is the Hadoop database, a distributed, scalable, big data store \""
date:       2019-07-20 02:30:00
author:     "YaPi"
header-img: "img/bigData/hbase_logo_with_orca_large.png"
tags:
    - 大数据
---


## 优化策略
什么导致HBase性能下降?

- jvm内存分配和GC回收策略
- HBase运行机制相关的部分配置不合理
- 表机构设计及用户使用方式不合理

##### HBase存储时耗时操作

1. HBase写入时当memstore达到一定的大小会flush到磁盘
保存成HFile,当HFile小文件太多会执行compact操作进行合并(当每一个store只包含一个HFile时,查询效率最高,避免了内存寻址时间)



2. 当Region的大小达到某一个阀值会执行split操作



- minor compaction: 选取一些小的、相邻的storeFile将他们合并成一个更大的StoreFile
    1. MemStore被flush到磁盘
    2. 用户执行shell命令compact major_compact或者调用相应的api
    3. HBase后台现场周期性触发检查
- major compaction: 将所有的StoreFile合并成一个StoreFile,清理无意义数据: 被删除的数据、TTL过期数据、版本号超过设定版本号的数据
- split: 当一个region达到一定的大小就会自动split成两个region


### 优化策略类别

- 常见服务端配置优化
- 常用优化策略(以实际需求、比如建表等)
- HBase读/写性能优化


#### 服务端优化

- JVM设置与GC设置
- hbase-site.xml部分属性设置



HBase properties | 简介
---|---
hbase.regionserver.handler.count | rpc请求的线程数量,默认是10
hbase.hregion.max.filesize | 当region的大小大于设定值后,hbase会开始split,默认是10G
hbase.hregion.majorcompaction | major compaction的执行周期,默认是一天,建议关闭,手动执行
hbase.hstore.compation.min | 一个store里的storefile总数超过该值,会触发合并操作
hbase.hstore.compaction.max | 一次最多合并多少storefile,根据内存设置,避免文件过多造成内存溢出
hbase.hstore.blockingStoreFiles | 一个 region中的Store(Coulmn Family)内有超过xx个storefile时,则block所有的写请求进行compaction
hfile.block.cache.size | regionserver的block cache的内存大小限制,根据实际去设置
hbase.hregion.memstore.flush.size | memstore超过该值将被flush
hbase.hregion.memstore.block.multiplier | 如果memstore的内存超过flush.size * multiplier,会阻塞该memstore的写操作


#### 常用的优化策略

主要分四个部分

- 预先分区
- RowKey优化
- Column优化
- Schema优化

##### 预先分区

HBase在创建表的时候,会自动创建一个Region分区,数据写入的时候,会直接写入,当数据量达到一定大小的时候,会执行split操作,进行分区,split分区非常耗时。

解决:

创建HBase表的时候预先创建一些空的Region,可以将频繁访问的数据放在一个或多个Region中,另外的放在另外的Region中,避免数据倾斜的问题


##### RowKey优化

- 利用HBase默认排序的特点,将一起访问的数据放到一起
- 防止热点问题(在分布式系统中,大量client集中访问某一个节点),避免使用时序或者单调递增递减等,一般通过加盐方式,在RowKey前面添加随机数

##### Column 优化

- 列族的名称和列的描述命名尽量简短
- 同一张表中ColumnFmaily的数量不要操作3个

##### Schema 优化

- 宽表: 一种"列多行少"的设计
- 高表: 一种"列少行多"设计

查询效率 : 高表大于宽表,查询的条件可以防止RowKey中,缓存的话,可以缓存更多的行
源数据的开销 : 行比较多,RowKey比较多,Region就也会比较多
事物性 : 宽表大于高表,因为HBase是单行事物

#### 读写性能的优化策略

##### 写流程

- 同步批量提交 or 异步批量提交
- WAL优化,是否必须,持久化等级,对于数据丢失是否能忍受,能就关闭WAL。会影响数据的完整性

##### 读性能

- 客户端 : Scan缓存设置,批量获取(不要同时请求太多,客户端可分批次请求)避免服务端带宽过大,本地内存溢出等问题
- 服务端 : BlockCache配置是否合理,HFile是否过多,过多是否需要执行compact命令进行合并
- 表结构设计


### HBase Coprecessor

- HBase协处理器受BigTable协处理器的启发,为用户提供类库和运行时环境,使得代码能够在HBase RegionServer和Master上处理
- 分为系统协处理器和表协处理器
    1. 系统协处理器: 全局加载到RegionServer托管的所有表和Region上
    2. 表协处理器: 用户可以指定一张表使用协处理器
- Observer and Endpoint
    1. Observer: 观察者,类似与触发器
        1. RegionObserver: 提供客户端的数据操纵事件钩子: Get、Put、Delete、Scan等
        2. MasterObserver: 提供DDL类型的操作钩子。如创建、删除、修改数据库表等
        3. WALObserver: 提供WAL相关操作钩子
    2. Endpoint: 终端,类似存储过程


#### Observer 应用场景

- 安全性: 例如执行Get或Put操作前,痛殴preGet或prePut方法检查是否允许该操作
- 引用完整性约束: HBase并不支持关系型数据库中的引用完整性约束概念,即通常所说的外键。我们可以使用协处理器增强这种约束。
- 二级索引: 可以使用协处理器来维持一个二级索引

#### Endpoint

- Endpoint是动态RPC插件的接口,它的实现代码被安装在服务器端,从而能够通过HBase RPC唤醒
- 调用接口,他们的实现代码会被目标RegionServer远程执行
- 典型的案例: 一个大的Table有几百个Region,需要计算某列的平均值或者总和

### 实战

#### Observer

```
import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.CellUtil;
import org.apache.hadoop.hbase.CoprocessorEnvironment;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.coprocessor.BaseRegionObserver;
import org.apache.hadoop.hbase.coprocessor.ObserverContext;
import org.apache.hadoop.hbase.coprocessor.RegionCoprocessorEnvironment;
import org.apache.hadoop.hbase.regionserver.wal.WALEdit;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.hadoop.hbase.util.CollectionUtils;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

/**
 * <p>
 *
 * </p>
 *
 * @author guazi
 * Email guazi@uoko.com
 * created by 2019/07/22
 */
public class RegionServerTest extends BaseRegionObserver {

    private byte[] columnFamily = Bytes.toBytes("cf");

    private byte[] countCol = Bytes.toBytes("countCol");

    private byte[] unDeleteCol = Bytes.toBytes("unDeleteCol");

    private RegionCoprocessorEnvironment environment;

    // RegionServer 打开region时执行
    @Override
    public void start(CoprocessorEnvironment e) throws IOException {
        super.start(e);
    }
    // RegionServer 关闭region时执行
    @Override
    public void stop(CoprocessorEnvironment e) throws IOException {
        super.stop(e);
    }

    /**
     * 1. cf : countCol 进行累加操作 每次插入时,都要与之前的值进行相加
     */
    @Override
    public void prePut(ObserverContext<RegionCoprocessorEnvironment> e, Put put, WALEdit edit, Durability durability) throws IOException {

        if (put.has(columnFamily,countCol)){
            // 如果发现新插入的包含countCol这一列
            // 获取old countCol value
            Result rs = e.getEnvironment().getRegion().get(
                    new Get(put.getRow())
            );

            int oldNum = 0;

            for (Cell cell:rs.rawCells()){
                if (CellUtil.matchingColumn(cell,columnFamily,countCol)){
                    oldNum = Integer.valueOf(Bytes.toString(CellUtil.cloneValue(cell)));
                }
            }

            // 获取new countCol value
            List<Cell> cells = put.get(columnFamily,countCol);
            int newNum = 0;
            for (Cell cell : cells){
                if (CellUtil.matchingColumn(cell,columnFamily,countCol)){
                    newNum = Integer.valueOf(Bytes.toString(CellUtil.cloneValue(cell)));
                }
            }

            // sum AND update
            put.addColumn(columnFamily,countCol,Bytes.toBytes(
                    String.valueOf(oldNum + newNum)
            ));
        }
    }

    /**
     * 2. 不能直接删除unDeleteCol 删除countCol的时候将unDeleteCol一起删除
     */
    @Override
    public void preDelete(ObserverContext<RegionCoprocessorEnvironment> e,
                          Delete delete, WALEdit edit, Durability durability) throws IOException {
        // 判断是否操作cf列
        List<Cell> cells = delete.getFamilyCellMap().get(columnFamily);

        if (CollectionUtils.isEmpty(cells))return;

        boolean deleteFlag = false;

        for (Cell cell : cells) {
            byte[] qualifier = CellUtil.cloneQualifier(cell);

            if (Arrays.equals(qualifier, unDeleteCol)) {
                throw new IOException("can not delete unDel column");
            }
            if (Arrays.equals(qualifier, countCol)) {
                deleteFlag = true;
            }
        }

        if (deleteFlag){
            delete.addColumn(columnFamily,unDeleteCol);
        }
    }
}

```

#### Endpoint

1. 定义相应的protobuffer接口
2. 生成相关代码
3. 编写自己要实现的业务代码

```
public class TestRowCountEndPoint extends hbaseEndPointTestService
                implements Coprocessor, CoprocessorService {

                @Override
                ...
}
```


#### 加载

将协处理器加载到hbase

- 配置文件加载: 即通过hbase-site.xml文件配置加载,一般这样的协处理器是系统级别的
- shell加载: 可以通过alter命令来对表进行schema修改来加载协处理器
- 通过API代码加载: 即通过API的方式来加载协处理器

1、3都需要将代码jar包防止hbase的classpath当中
2. 只需要上传到hbase的集群文件就行了


##### 配置文件加载

value 为类名全路径

属性 | 说明
---|---
hbase.coprocessor.region.clasees | 配置RegionObservers和Endpoints
hbase.coprocessor.wal.classes | 配置WALObservers
hbase.coprocessor.master.classes | 配置MasterObservers

##### shell加载

alter 'CoprocessorTestTable','coprocessor'=>
'hdfs://localhost:9000/path/coprocessor.jar|com.immoc.hbase.TestRegionObserver|1001|'

> 参数 1. 路径 2. 全路径名 3. 优先级




#### HBase 备份与恢复

- CopyTable (基于MapReduce)
- Export/Import (基于MapReduce)
- Snapshot
- Replication (常用)

##### CopyTable

- 支持时间区间、row区间,改变表名称,改变列族名称,指定是否copy已经被删除的数据等功能
- CopyTable工具采用scan查询,写入新表时采用put和delete API,全是基于hbase的client api进行读写

```
1. 创建新的表并指定相应列族
2. 执行命令进行拷贝

./hbase org.apache.hadoop.hbase.mapreduce.CopyTable --new.name=FileTableNew FileTable

```

#### Export/Import

- Export可导出数据到目标集群,然后可在目标集群Import导入数据,Export支持指定开始时间和结束时间,因此可以做增量备份
- Export导出工具与CopyTable一样是依赖hbase的scan读取数据

```
Export

./hbase org.apache.hadoop.hbase.mapreduce.Export {导出的表名} {存储在hdfs的路径}

Import

./hbase org.apache.hadoop.hbase.mapreduce.Import {导入的表名} {存储在hdfs的路径}

```

#### Snapshot

- Snapshot(快照)作用于表上。通过配置hbase-site.xml开启该功能
- 可以快速的恢复表至快照指定的状态从而迅速的修复数据(会丢失快照之后的数据)

```
<property>
    <name>hbase.snapshot.enabled</name>
    <value>true</value>
</property>
```

- 使用

```
#创建快照
snapshot 'myTable','myTableSnapshot-190731'
#克隆快照
clone_snapshot 'myTableSnapshot-190731','myNewTestTable'
#列出快照
list_snapshots
#删除快照
delete_snapshot 'myTableSnapshot-190731'
#恢复数据
disable 'myTable'
restore_snapshot 'myTableSnapshot-190731'
```

#### HBase容灾策略

- 可以通过replication机制实现hbase集群的主从模式
- replication是依赖WAL日志进行的同步(若没写WAL日志,replication是不生效的)

```
hbase-site.xml

<property>
    <name>hbase.replication</name>
    <value>true</value>
</property>
```

- 使用

```
#在原集群及目标集群都创建同名表
#指定目标集群zk地址和路径

# 执行命令
add_peer '1',"zk01:2181:/hbase_backup"

#标注需要备份的列族信息及备份的目标库地址
# replication_scope值为上面add_peer指定的值

alter 'replication_source_table',{NAME=>'f1',REPLICATION_SCOPE=>'1'}
```