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

#### 与Memcached的异同

相同点：

- 都是基于内存的数据库，一般都用来当做缓存使用
- 都有过期策略

异同点：

- Redis 支持更丰富的数据类型(支持更复杂的应用场景)。Redis 不仅仅支持简单的 k/v 类 型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。Memcached 只支持最简 单的 k/v 数据类型
- Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用,而 Memecache 把数据全部存在内存之中
- Redis 在服务器内存使用完之后，可以将不用的数据放到磁盘上。但是，Memcached在服务器内存使用完之后，就会直接报异常
- Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据;但是Redis 目前是原生支持 cluster 模式的
- Memcached 是多线程，非阻塞IO复用的网络模型;Redis 使用单线程的多路 IO 复用模型(Redis 6.0 引入了多线程 IO )
- Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持。并且，Redis支持更多的编程语言
- Memcached过期数据的删除策略只用了惰性删除，而 Redis 同时使用了惰性删除与定期删除

#### 特点
Redis 基于 Reactor 模式来设计开发了自己的一套高效的事件处理模型 (Netty 的线程模型也基 于 Reactor 模式，Reactor 模式不愧是高性能 IO 的基石)，这套事件处理模型对应的是 Redis 中的文件事件处理器(file event handler)。
由于文件事件处理器(file event handler)是单线程 方式运行的，所以我们一般都说 Redis 是单线程模型

Redis 通过IO 多路复用程序 来监听来自客户端的大量连接(或者说是监听多个 socket)，它会
将感兴趣的事件及类型(读、写)注册到内核中并监听每个事件是否发生。
这样的好处非常明显: I/O 多路复用技术的使用让 Redis 不需要额外创建多余的线程来监听客户
端的大量连接，降低了资源的消耗(和 NIO 中的 Selector 组件很像)。
另外， Redis 服务器是一个事件驱动程序，服务器需要处理两类事件: 1. 文件事件; 2. 时间事
件

Redis 基于 Reactor 模式开发了自己的网络事件处理器:这个处理器被称为文件事件处理器 (file event handler)。文件事件处理器使用 I/O 多路复用(multiplexing)程序来同时监听 多个套接字，并根据 套接字目前执行的任务来为套接字关联不同的事件处理器。
当被监听的套接字准备好执行连接应答(accept)、读取(read)、写入(write)、关 闭 (close)等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套 接字之前关联好的事件处理器来处理这些事件。
虽然文件事件处理器以单线程方式运行，但通过使用 I/O 多路复用程序来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型，又可以很好地与 Redis 服务器中其他同样 以单线程方式运行的模块进行对接，这保持了 Redis 内部单线程设计的简单性


Redis6.0 引入多线程主要是为了提高网络 IO 读写性能，因为这个算是 Redis 中的一个性能瓶颈
(Redis 的瓶颈主要受限于内存和网络)。

虽然，Redis6.0 引入了多线程，但是 Redis 的多线程只是在网络数据的读写这类耗时操作上使用了， 执行命令仍然是单线程顺序执行。因此，你也不需要担心线程安全问题。
Redis6.0 的多线程默认是禁用的，只使用主线程。如需开启需要修改 redis 配置文件 :

```text
io-threads-do-reads yes

io-threads 4 #官网建议4核的机器建议设置为2或3个线程，8核的建议设置为6个线程
```
开启多线程后，还需要设置线程数，否则是不生效的。同样需要修改 redis 配置文件

#### 问题

##### 过期的数据的删除策略了解么?
如果假设你设置了一批 key 只能存活 1 分钟，那么 1 分钟后，Redis 是怎么对这批 key 进行删除
的呢?
常用的过期数据的删除策略就两个

1. 惰性删除 :只会在取出key的时候才对数据进行过期检查。这样对CPU最友好，但是可能会 造成太多过期 key 没有被删除
2. 定期删除 : 每隔一段时间抽取一批 key 执行删除过期key操作。并且，Redis 底层会通过限 制删除操作执行的时⻓和频率来减少删除操作对CPU时间的影响


定期删除对内存更加友好，惰性删除对CPU更加友好。两者各有千秋，所以Redis 采用的是 定期 删除+惰性/懒汉式删除 。
但是，仅仅通过给 key 设置过期时间还是有问题的。因为还是可能存在定期删除和惰性删除漏掉 了很多过期 key 的情况。这样就导致大量过期 key 堆积在内存里，然后就Out of memory了。
怎么解决这个问题呢?答案就是: Redis 内存淘汰机制

##### Redis 内存淘汰机制了解么?
比如：
MySQL 里有 2000w 数据，Redis 中只存 20w 的数据，如何保证 Redis 中的数 据都是热点数据?

Redis 提供 6 种数据淘汰策略:
1. volatile-lru(least recently used):从已设置过期时间的数据集(server.db[i].expires) 中挑选最近最少使用的数据淘汰
2. volatile-ttl:从已设置过期时间的数据集(server.db[i].expires)中挑选将要过期的数据淘汰
3. volatile-random:从已设置过期时间的数据集(server.db[i].expires)中任意选择数据淘汰
4. allkeys-lru(least recently used):当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key(这个是最常用的)
5. allkeys-random:从数据集(server.db[i].dict)中任意选择数据淘汰
6. no-eviction:禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧!
7. volatile-lfu(least frequently used):从已设置过期时间的数据集(server.db[i].expires)中 挑选最不经常使用的数据淘汰
8. allkeys-lfu(least frequently used):当内存不足以容纳新写入数据时，在键空间中，移 除最不经常使用的 key
