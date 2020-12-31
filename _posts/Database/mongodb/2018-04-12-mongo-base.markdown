---
layout:     post
title:      "MongoDB基础知识点"
subtitle:   ""
date:       2018-04-12 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - mongodb
---

#### 基础知识点

##### 规范

###### 数据库规范

数据库名称要求满足如下特征：

- 不能是空字符串("")
- 不得含有‘’、空格、. 、$、/、\、\0(空字符)
- 应该全部小写
- 最多64字节
- 不得是保留数据库名(admin、config、local)

MongoDB中默认的数据库为test,如果没有创建新的数据库，集合将存放在test数据中

```
查看数据库版本: db.version()
```

###### 集合、文档规范

集合命名遵循如下规则：
- 集合名不能是空字符串
- 集合名不能有\0字符，这个字符表示集合名的结尾
- 集合名不能以system.开头，这是为系统保留的前缀
- 用户创建的集合名字不能含有保留字符。有些驱动程序支持，这是因为某些系统生成的集合中包含该字符。
- 文档的键是字符串。除了少数例外情况，可以使用任意UTF-8字符
- 文档不能有重复的键
- 区分大小写
- 文档的值不仅可以是字符串，还可以是其他数据类型，甚至是文档，且键值对是有序的


##### 数据库相关命令

```
选择某一个数据库
use 数据库名

若想新建某个数据库，需要首先use数据库，再创建相关集合，否则不生效

显示所有数据库
show dbs

删除数据库
首先切换到相应数据库，然后执行 db.dropDatabase()，注意此数据库中若有集合则不生效
```

##### 集合、文档相关命令

###### 插入文档

1.8版本定义boson最大为16M

```
db.col.save(document) 命令。如果不指定 _id 字段 save() 方法类似于 insert() 方法。如果指定 _id 字段，则会更新该 _id 的数据


3.2 版本后还有以下几种语法可用于插入文档:
db.collection.insertOne():向指定集合中插入一条文档数据
db.collection.insertMany():向指定集合中插入多条文档数据

```
###### 更新文档

1. 更新修改器： $inc
    1. 使用修改器_id不能改变，其他的都可以改变
db.yp.update({url:"ww.baidu.com"},{"$inc":{"pageviews":1}})

2. $set 用来指定一个键的值。如果不存在则创建它。
    1. $inc 和$set 在指定字段时若不存在则会增加，$inc只能用于整数、长整数、双精度数。$set 可以改变原来键的类型
3. $push会向已有的数组末尾加一个元素，没有则新建数组
4. $addToSet 判断是否存在，不存在才新增 $ne 有同样功能，但有时有限制
5. {$pop :{key:1}}从数组末尾删除，key:-1从头部删除s
6. 依据特定条件删除元素 $pull
    1.  db.yp.update({x:1},{$pull:{"todo":"laundry"}) 删除todo里面为laundry元素
7. 可以运用数组下标进行定位数组
    1. db.yp.update({"comments.author":"jogn"},{$set:{"comments.$.author":"jim"}}),更新第一条找到的数据
8. $or 查询多个限定条件
9. $mod 会将查询的值除以第一个给定的值日，若余数等于第二个给定值则返回结果，db.yp.find({"id_num":{"$not":{“$mod":[5,1]}}})
10. 一个键可以有多个条件，但是不能有多个更新修改器，$inc $set
11. 查询特定条件，键值为null
    1. db.yp.find({"z":{"$in":[null],"$exists":true}})
12. db.yp.find({"comment":{"$elemMatch":{"autor":"joe","score":{"$get":5"}}}"})
13.

```
db.col.update({'title':'MongoDB 教程'},{$set:{'title':'MongoDB'}})
以上语句只会修改第一条发现的文档，如果你要修改多条相同的文档，则需要设置 multi 参数为 true。

db.col.update({'title':'MongoDB 教程'},{$set:{'title':'MongoDB'}},{multi:true})

或

db.col.update( { "count" : { $gt : 5 } } , { $set : { "test5" : "OK"} },true,true )

第三个参数，若不存在执行查询条件数据就新增
第四个参数，操作全部数据，全部执行更新操作

在3.2版本开始，MongoDB提供以下更新集合文档的方法：
db.collection.updateOne() 向指定集合更新单个文档
db.collection.updateMany() 向指定集合更新多个文档
```



###### 删除文档

```
db.col.remove({'title':'MongoDB 教程'})

如果你只想删除第一条找到的记录可以设置 justOne 为 1，如下所示：
db.testCollection.remove({"name:"123},1)

remove() 方法已经过时了，现在官方推荐使用 deleteOne() 和 deleteMany() 方法。

如删除集合下全部文档：
db.inventory.deleteMany({})
删除 status 等于 A 的全部文档：
db.inventory.deleteMany({ status : "A" })
```

##### 索引

###### 创建索引
需要对进行排序的键创建索引，若不创建，则进行排序的数据不能做T级别的排序，否则会报错

创建索引可以对索引进行命名
db.yp.ensureIndex({a:1,b:1....;},{"name":"yaerluo"})

唯一索引确保文档指定键都有唯一的值
db.yp.ensureIndex({"username":1},{"unique":true})


消除重复
db.yp.ensureIndex({"username":1},{"unique":true,"dropDups":true}) 删除除了第一个文档外的文档

索引管理 存在system.indexes集合中，只能通过ensureIndex和dropIndexes进行
添加索引时，可以在后台进行，否则会阻断建立索引期间所有请求
db.yp.ensureIndex({"username":1},{"background":true})

删除索引，将x换为*
db.runCommand({"dropIndexes":"集合名（yp）","index":"x":})
删除所有索引，将x换为*


创建地理空间索引
db.yp.ensureIndex({"gps":"2d"})
它的值必须是一个包含两个元素的数组或者包含两个键的内嵌文档  大小默认为-180 到180
###### 简单查询

```
美化返回数据，使返回的数据进行格式化
db.col.find().pretty()
pretty()方法会以格式化的方式来显示所有文档

db.col.find({"likes": {$gt:50}, $or: [{"by": "菜鸟教程"},{"title": "MongoDB 教程"}]}).pretty()


排序
在MongoDB中使用使用sort()方法对数据进行排序，sort()方法可以通过参数指定排序的字段，并使用 1 和 -1 来指定排序的方式，其中 1 为升序排列，而-1是用于降序排列
```

MongoDB支持的条件操作符

意义 | MongoDB
---|---
大于 | $gt
小于 | $lt
大于等于 | $gte
小于等于 | $lte
不等于| $ne
等于 | $eq

###### 复合查询 aggregate

例：
```
db.articles.aggregate( [
                        { $match : { score : { $gt : 70, $lte : 90 } } },
                        { $group: { _id: null, count: { $sum: 1 } } }
                       ] );
```

- $project：修改输入文档的结构。可以用来重命名、增加或删除域，也可以用于创建计算结果以及嵌套文档。
- $match：用于过滤数据，只输出符合条件的文档。$match使用MongoDB的标准查询操作。
- $limit：用来限制MongoDB聚合管道返回的文档数。
- $skip：在聚合管道中跳过指定数量的文档，并返回余下的文档。
- $unwind：将文档中的某一个数组类型字段拆分成多条，每条包含数组中的一个值。
- $group：将集合中的文档分组，可用于统计结果。
- $sort：将输入文档排序后输出。
- $geoNear：输出接近某一地理位置的有序文档。

```
按日、按月、按年、按周、按小时、按分钟聚合操作如下：
db.getCollection('m_msg_tb').aggregate(
[
    {$match:{m_id:10001,mark_time:{$gt:new Date(2017,8,0)}}},
    {$group: {
       _id: {$dayOfMonth:'$mark_time'},
        pv: {$sum: 1}
        }
    },
    {$sort: {"_id": 1}}
])

时间关键字如下：
 $dayOfYear: 返回该日期是这一年的第几天（全年 366 天）。
 $dayOfMonth: 返回该日期是这一个月的第几天（1到31）。
 $dayOfWeek: 返回的是这个周的星期几（1：星期日，7：星期六）。
 $year: 返回该日期的年份部分。
 $month： 返回该日期的月份部分（ 1 到 12）。
 $week： 返回该日期是所在年的第几个星期（ 0 到 53）。
 $hour： 返回该日期的小时部分。
 $minute: 返回该日期的分钟部分。
 $second: 返回该日期的秒部分（以0到59之间的数字形式返回日期的第二部分，但可以是60来计算闰秒）。
 $millisecond：返回该日期的毫秒部分（ 0 到 999）。
 $dateToString： { $dateToString: { format: , date: } }。
```

