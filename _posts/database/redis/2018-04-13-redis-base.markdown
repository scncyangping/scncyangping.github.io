---
layout:     post
title:      "Redis基础命令及内存划分"
subtitle:   ""
date:       2018-04-13 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - redis
---

#### 基本数据类型
##### string
String类型是包含很多种类型的特殊类型，并且是二进制安全的。比如序列化的对象进行储存，比如一张图片进行二进制储存，比如一个简单的字符串数值等等

set和get方法：
设置值set  name realValue
取值 get name
说明：set设置name的时候，如果name重复，设置的值会进行覆盖。

setnx 方法
设置值setnx name realValue
说明：如果这个name已经存在，不会进行覆盖，直接返回0.如果name不存在才会插入新的值。

setex方法
设置值setex name time(秒) realValue
说明：设置这个name的value在缓存中存在的过期时间，过了这个时间后返回nil。在redis中nil标示null的意思。

setrange方法：替换字符串
set email 123456@qq.com
setrange email 10 ww   表是从第几位开始替换成后面的字符串。
说明：此时把123456@qq.com替换成123456@qq.wwm

##### Hash
Hash类型是String类型的filed和value的映射表，特别适合存储对象。相比较而言把一个对象存储在Hash类型中要比直接存储在String中更加节省空间。并方便存储整个对象，Hash类型也是我们工作中最常用的一种

形如：hset user name ming  意思是一个Hash类型叫做user，这个user的属性name的值是ming。

使用hget来获取值   hget user  name 就能获取到这个对象中的name属性的值。

hmset可进行批量存储多个键值对。hmset user age 15 sex man

hmget可进行批量获取多个键值对。hmget user name age sex

Hash类型中同样也有hsetnx，他和setnx大同小异。

hincrby和hdecrby集合递增和递减。

hexists 如果存在返回1，不存在返回0

hlen 返回hash中所有键的数值。

hkeys返回hash中的所有键。

hvals 返回Hash中所有的值。

hgetall返回Hash中所有的键和值。


##### List
List类型是一个链表结构的集合，其主要功能有push，pop获取元素等等。更详细的说List类型是一个双端链表结构，我们可以通过相关操作进行集合的头部或者尾部添加删除元素。List的设计非常简单精巧，既可以作为栈又可以作为队列。满足绝大多数要求

lpush方法：从头部添加元素，（栈）先进后出。
设置值 lpush list hello
说明：创建一个name为list的栈，并且入栈一个hello

rpush方法：从尾部添加元素（队列）先进先出
设置值lpush list2  hello
说明：创建一个name为list2的队列，并且入栈一个hello

lrange方法：查看list中的值

linsert list2 before [集合的元素] [要插入的元素]

lset方法  将指定下标的元素替换掉

lrem方法：删除制定元素，并且返回删除元素的个数。

lpop方法：从List头部删除元素，并且返回删除的元素。

rpop方法：从List尾部删除元素，并且返回删除的元素。

llen方法：返回元素的个数。

lindex方法：返回名称为key的元素在List中的index位置的元素。lindex  list2 0 返回第一个元素

##### set
set集合是String类型的无序集合，set是通过hashtable实现的，对集合我们可以取交集，并集，差集

sadd方法：向名称为key的set中添加元素。
小结：set集合不允许重复元素，smembers查看set中的所有元素。

srem方法  删除set集合元素。srem name 值

spop方法 随机返回删除的key

sdiff返回两个集合不同元素，哪个集合在前面就以哪个集合为标准。

sdiffstore 将返回的不同元素存储在另一个集合里面。 sdiffstore set3 set1 set2 。吧1和2的不同元素存储在3中

sinter 返回两个集合的交集。sinter set1 set2 返回set1中和set2中的交集元素。

sinterstore 将返回的交集存储在一个新的集合中

smove方法：从一个set集合中移动元素到另一个set集合中 smove set2 set1 bbb 将set2中的bbb移动到set1中。

scard方法：查看集合中元素的个数

##### ZSet
Zset是在set的基础上做了一个有序的调整。

zadd方法：向有序集合中添加一个元素，如果该元素存在，就更新顺序。
小结：在重复插入的时候会根据顺序属性更新。

语法：zadd set1 1 aaa   其中的1代表序号。 就是排序的序号。aaa代表集合的值，set1代表集合的名字。

zrange 方法，查看集合中的值 zrange set1 0 -1 withscores
说明：withscores代表把序号也查询出来，不想显示序号可以不加。

zrem方法  删除集合中的元素。


##### redis高级命令
keys * 返回所有的name

exists 是否存在指定的name

expire 设置某个key的过期时间，使用ttl查看剩余时间

persist 取消过期时间

select选择数据库，数据库为0到15，共16个数据库，默认进入的是0个数据库。

move key [数据库下标] 转移到其他数据库中

randomkey  随机返回数据库中的一个key

rename key newkey 重命名key

dbsize 查看当前数据库中key的数量

flushdb 清空当前数据库，flushall清空所有数据库。

config get * 获取当前redis配置项。

info 获取数据库信息。


[https://www.cnblogs.com/jpfss/p/10224436.html](https://www.cnblogs.com/jpfss/p/10224436.html)



#### 内存划分

##### 数据

Redis使用键值对存储数据，其中的值（对象）包括5种类型，即字符串、哈希、列表、集合、有序集合。这5种类型是Redis对外提供的，实际上，在Redis内部，每种类型可能有2种或更多的内部编码实现；此外，Redis在存储对象时，并不是直接将数据扔进内存，而是会对对象进行各种包装：如redisObject、SDS等

##### 进程本身运行需要的内存

Redis主进程本身运行肯定需要占用内存，如代码、常量池等等；这部分内存大约几兆，在大多数生产环境中与Redis数据占用的内存相比可以忽略。这部分内存不是由jemalloc分配，因此不会统计在used_memory中。

补充说明：除了主进程外，Redis创建的子进程运行也会占用内存，如Redis执行AOF、RDB重写时创建的子进程。当然，这部分内存不属于Redis进程，也不会统计在used_memory和used_memory_rss中。

##### 缓冲内存
缓冲内存包括客户端缓冲区、复制积压缓冲区、AOF缓冲区等；其中，客户端缓冲存储客户端连接的输入输出缓冲；复制积压缓冲用于部分复制功能；AOF缓冲区用于在进行AOF重写时，保存最近的写入命令。

##### 内存碎片
内存碎片是Redis在分配、回收物理内存过程中产生的。例如，如果对数据的更改频繁，而且数据之间的大小相差很大，可能导致redis释放的空间在物理内存中并没有释放，但redis又无法有效利用，这就形成了内存碎片。内存碎片不会统计在used_memory中