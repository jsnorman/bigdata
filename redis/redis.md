# 缓存中间件--Redis与Memcache的区别？

* Memchache: 代码层次类似Hash
  * 支持简单数据类型
  * 不支持数据持久化存储
  * 不支持主从
  * 不支持分片
* Redis
  * 数据类型丰富
  * 支持数据磁盘持久化存储
  * 支持主从
  * 支持分片

# 为什么Redis能这么快?

单点TPS达到8万/秒，QPS(query per second) 每秒查询次数达到10万/秒

* 完全基于内存,绝大部分请求是纯粹的内存操作,执行效率高
* 数据结构简单,对数据操作也简单
* 采用单线程,单线程也能处理高并发请求,想多核也可启动多实例
* 使用多路I/O复用模型,非阻塞IO

## 多路I/O复用模型

* FD : File Descriptor,文件描述符

  一个打开的文件通过唯一的描述符进行引用,该描述符是打开文件的元数据到文件本身的映射]

  

# Redis支持的数据类型

1.string：最基本的数据类型，二进制安全的字符串，最大512M。

2.list：按照添加顺序保持顺序的字符串列表。

3.set：无序的字符串集合，不存在重复的元素。

4.sorted set：已排序的字符串集合。

5.Hash：key-value对的一种集合。

![img](https://pic1.zhimg.com/80/v2-9f60b2278b9cedc0aaae7c22d2a02b34_hd.jpg)

# 从海量Key里查询某一固定前缀的Key

**使用keys对线上的业务的影响**

  * KEYS pattern

    ```redis
    keys k1*
    ```

    * 查询所有符合给定模式pattern的key

    * KEYS指令一次性返回所有匹配的key

    * 键的数量过大会造成服务器卡顿

*  SCAN cursor

   ```redis
   scan 0 match k1* count 10    
   ```

   *  基于游标的迭代器,需要基于上一次的游标延续之前的迭代过程
   *  以0作为粮票开始一次新的迭代,直到命令返回游标0完成一次遍历
   *  不保证每次执行都返回某个给定数量的元素,支持模糊查询
   *  一次返回的数量不可控.只能是大概率符合count参数

# Redis 怎么实现分布式锁？

* 分布式锁需要解决的问题
  * 互斥性
  * 安全性
  * 死锁
  * 容错
* SETNX key value
  * 如果key不存在,则创建并赋值
  * 时间复杂度O(1)
  * 返回值
    * 设置成功.返回1
    * 设置失败,返回0

## 如何解决SETNX长期有效的问题?

* EXPIRE key seconds
  * 设置key的生存时间.当key过期时(生存时间为0),会被自动删除

## 最终方案

```
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```

* EX second 

  * 设置键的过期时间为second秒
* PX millisecond
  * 设置键的过期时间为millisecond 毫秒
* NX
  * 只在键不存在时,才对键进行设置操作
* XX
  * 只在键已经存在时,才对键进行设置操作
* SET操作成功完成时,返回OK,否则返回nil

# 大量的key同时过期的注意事项

* 集中过期,由于清除大量的key很耗时,会出现短暂的卡顿现象

* 解决:在设置key过期时间的时候,给每个key加上随机值

# 如何使用Redis做异步队列?

* **使用List作为队列,RPUSH生产消息,LPOP消费消息**
  * 缺点
    * 没有等待队列里有值就直接消费
  * 弥补
    * 可以通过在应用层引用Sleep机制去调用LPOP重试

* **BLPOP** **key** [**key** ...] **timeout** :**阻塞直到队列有消息或者超时**

  ```
  blpop testlist 30   //30秒内一直等待
  ```

  * 缺点
    * 只能供一个消费者消费

* **pub/sub:主题订阅者模式**

  * 发送者(pub)发送消息,订阅者(sub接收消息)

  * 订阅者可以订阅任意数量的频道(topic)

    * > publish myTopic "Hello"

    * > subscribe myTopic

  * 缺点

    * 消息的发布是无状态的,无法保证可达

# Redis如何做持久化?

* RDB(快照)持久化
  * 保存某个时间点的全量数据快照
  * SAVE
    * 阻塞Redis的服务进程,直到RDB文件被创建完毕
  * BGSAVE
    * Fork出一个子进程来创建RDB文件,不阻塞服务器进程
  * 缺点
    * 内存数据的全量同步,数据量大会由于I/O而严重影响性能
    * 可能会因为Redis挂掉而丢失从当前至最近一次快照期间的数据
  * 自动触发RDB持久化的方式
    * 根据redis.conf配置里的SAVE m n 定时触发(用的是BGSAVE)
    * 主从复制时,主节点自动触发
    * 执行Debug Reload
    * 执行Shutdown且没有开启AOF
* AOF(Append-Only-File)持久化:保存写状态
  * 记录下除了查询以外的所有变更数据库状态的指令
  * 以append的形式追加保存到AOF文件(增量)
* RDB-AOF混合持久化方式
  * BGSAVE做镜像全量持久化,AOF做增量持久化

## 日志重写解决AOF文件大小不断增大的问题

* 调用fork(),创建一个子进程
* 子进程把新的AOF写到一个临时文件里,不依赖原来的AOF文件
* 主进程持续将新的变动同时写到内存和原来的AOF里
* 主进程获取子进程重写AOF的完成信号,往新AOF同步增量变动
* 使用新的AOF文件替换掉旧的AOF文件



# Redis数据的恢复

RDB和AOF文件共存情况下的恢复流程

* 检查是否有AOF文件
  * 若存在,则使用AOF文件
  * 否则,使用RDB文件

RDB和AOF的优缺点

* RDB优点
  * 全量数据快照,文件小,恢复快
* RDB缺点
  * 无法保存最近一次快照之后的数据
* AOF优点
  * 可读性高,适合保存增量数据,数据不易丢失
* AOF缺点
  * 文件体积大,恢复时间长

# 使用Pipeline的好处

* Pipeline和linux和管道类似
* Redis基于请求/响应模型,单个请求处理需要一一应答
* Pipeline批量执行指令,节省多次IO往返的时间
* 有顺序依赖的指令建议分批发送

# Redis的同步机制

* 全同步过程
  * Slave发送sync命令到Master
  * Master启动一个后台进程,将Redis中的数据快照保存到文件中
  * Master将保存数据快照期间接收到的写命令缓存起来
  * Master完成写文件操作后,将该文件发送给Slave
  * 使用新的AOF文件夫胸掉旧的AOF文件
  * Master 将这期间收集的增量写命令发送给Slave端
* 增量同步过程
  * Master接收到用户的操作指令,判断是否需要传播到Slave
  * 将操作记录追加到AOF文件
  * 将操作传播到其他Slave
    * 对齐主从库
    * 往响应缓存写入指令
  * 将缓存中的数据发送给Slave

# Redis Sentinel(哨兵)

解决主从同步Master宕机后的主从切换问题

* 监控
  * 检查主从服务器是否运行正常
* 提醒
  * 通过API向管理员或者其他应用程序发送故障通知
* 自动故障迁移
  * 主从切换

#  Redis主要有哪些功能？

**1.哨兵（Sentinel）和复制（Replication）**

Redis服务器毫无征兆的罢工是个麻烦事，如何保证备份的机器是原始服务器的完整备份呢？这时候就需要哨兵和复制。

哨兵Sentinel可以管理多个Redis服务器，它提供了监控，提醒以及自动的故障转移的功能，Replication则是负责让一个Redis服务器可以配备多个备份的服务器。

Redis也是利用这两个功能来保证Redis的高可用的。

**2.事务**

很多情况下我们需要一次执行不止一个命令，而且需要其同时成功或者失败。redis对事务的支持也是源自于这部分需求，即支持一次性按顺序执行多个命令的能力，并保证其原子性。

**3.LUA脚本**

在事务的基础上，如果我们需要在服务端一次性的执行更复杂的操作（包含一些逻辑判断），则lua就可以排上用场了。

**4.持久化**

redis的持久化指的是redis会把内存的中的数据写入到硬盘中，在redis重新启动的时候加载这些数据，从而最大限度的降低缓存丢失带来的影响。

**5.集群（Cluster）**

单台服务器资源的总是有上限的，CPU资源和IO资源我们可以通过主从复制，进行读写分离，把一部分CPU和IO的压力转移到从服务器上，这也有点类似mysql数据库的主从同步。

在Redis官方的分布式方案出来之前，有twemproxy和codis两种方案，这两个方案总体上来说都是依赖proxy来进行分布式的，**下面的内容有具体集群方案详解。**

## **Redis是单进程单线程的？**

Redis是单进程单线程的，Redis利用队列技术将并发访问变为串行访问，消除了传统数据库串行控制的开销。

## **Redis为什么是单线程的？**

多线程处理会涉及到锁，而且多线程处理会涉及到线程切换而消耗CPU。因为CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存或者网络带宽。**单线程无法发挥多核CPU性能，不过可以通过在单机开多个Redis实例来解决。**

##  **使用Redis的优势？**

1. 速度快，因为数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)

2. 支持丰富数据类型，支持string，list，set，sorted set，hash
3. 支持事务，操作都是原子性，所谓的原子性就是对数据的更改要么全部执行，要么全部不执行

4. 丰富的特性：可用于缓存，消息，按key设置过期时间，过期后将会自动删除

##  **Redis有哪几种数据淘汰策略？**

在Redis中，允许用户设置最大使用内存大小server.maxmemory，当Redis 内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。

1.volatile-lru:从已设置过期的数据集中挑选最近最少使用的淘汰

2.volatile-ttr:从已设置过期的数据集中挑选将要过期的数据淘汰

3.volatile-random:从已设置过期的数据集中任意挑选数据淘汰

4.allkeys-lru:从数据集中挑选最近最少使用的数据淘汰

5.allkeys-random:从数据集中任意挑选数据淘汰

6.noenviction:禁止淘汰数据

redis淘汰数据时还会同步到aof

## **Redis集群方案应该怎么做？都有哪些方案？**

1.twemproxy

2.codis，目前用的最多的集群方案，基本和twemproxy一致的效果，但它支持在 节点数量改变情况下，旧节点数据可恢复到新hash节点。

3.Redis cluster3.0自带的集，特点在于他的分布式算法不是一致性hash，而是hash槽的概念，以及自身支持节点设置从节点。

## **Redis读写分离模型**

通过增加Slave DB的数量，读的性能可以线性增长。为了避免Master DB的单点故障，集群一般都会采用两台Master DB做双机热备，所以整个集群的读和写的可用性都非常高。

读写分离架构的缺陷在于，不管是Master还是Slave，每个节点都必须保存完整的数据，如果在数据量很大的情况下，集群的扩展能力还是受限于单个节点的存储能力，而且对于Write-intensive类型的应用，读写分离架构并不适合。

## **Redis数据分片模型**

为了解决读写分离模型的缺陷，可以将数据分片模型应用进来。

可以将每个节点看成都是独立的master，然后通过业务实现数据分片。

结合上面两种模型，可以将每个master设计成由一个master和多个slave组成的模型。

## redis都有哪些使用场景？

[聊聊 Redis 使用场景](https://zhuanlan.zhihu.com/p/56154153)

随着数据量的增长，MySQL 已经满足不了大型互联网类应用的需求。因此，Redis 基于内存存储数据，可以极大的提高查询性能，对产品在架构上很好的补充。在某些场景下，可以充分的利用 Redis 的特性，大大提高效率。 

* 缓存 

**对于热点数据，缓存以后可能读取数十万次**，因此，对于热点数据，缓存的价值非常大。例如，分类栏目更新频率不高，但是绝大多数的页面都需要访问这个数据，因此读取频率相当高，可以考虑基于 Redis 实现缓存。 

* 会话缓存

 此外，还可以考虑使用 Redis 进行会话缓存。例如，将 web session 存放在 Redis 中。 

* 时效性 

例如验证码只有60秒有效期，超过时间无法使用，或者基于 Oauth2 的 Token 只能在 5 分钟内使用一次，超过时间也无法使用。 

* 访问频率 

出于减轻服务器的压力或防止恶意的洪水攻击的考虑，需要控制访问频率，例如限制 IP 在一段时间的最大访问量。 

* 计数器 

数据统计的需求非常普遍，通过原子递增保持计数。例如，应用数、资源数、点赞数、收藏数、分享数等。 

* 社交列表 

社交属性相关的列表信息，例如，用户点赞列表、用户分享列表、用户收藏列表、用户关注列表、用户粉丝列表等，使用 Hash 类型数据结构是个不错的选择。

*  记录用户判定信息 

记录用户判定信息的需求也非常普遍，可以知道一个用户是否进行了某个操作。例如，用户是否点赞、用户是否收藏、用户是否分享等。 

* 交集、并集和差集

 在某些场景中，例如社交场景，通过交集、并集和差集运算，可以非常方便地实现共同好友，共同关注，共同偏好等社交关系。 

* 热门列表与排行榜 

按照得分进行排序，例如，展示最热、点击率最高、活跃度最高等条件的排名列表。 

* 最新动态

 按照时间顺序排列的最新动态，也是一个很好的应用，可以使用 Sorted Set 类型的分数权重存储 Unix 时间戳进行排序。 

* 消息队列 

Redis 能作为一个很好的消息队列来使用，依赖 List 类型利用 LPUSH 命令将数据添加到链表头部，通过 BRPOP 命令将元素从链表尾部取出。同时，市面上成熟的消息队列产品有很多，例如 RabbitMQ。因此，更加建议使用 RabbitMQ 作为消息中间件。

## **Redis常见性能问题和解决方案？**

(1) Master最好不要做任何持久化工作，如RDB内存快照和AOF日志文件

(2) 如果数据比较重要，某个Slave开启AOF备份数据，策略设置为每秒同步一次

(3) 为了主从复制的速度和连接的稳定性，Master和Slave最好在同一个局域网内

(4) 尽量避免在压力很大的主库上增加从库

(5) 主从复制不要用图状结构，用单向链表结构更为稳定，即：Master <- Slave1 <- Slave2 <- Slave3...

这样的结构方便解决单点故障问题，实现Slave对Master的替换。如果Master挂了，可以立刻启用Slave1做Master，其他不变。

 

## **Redis支持的Java客户端都有哪些？官方推荐用哪个？**

Redisson、Jedis、lettuce等等，官方推荐使用Redisson。

## **Redis哈希槽的概念？**

Redis集群没有使用一致性hash,而是引入了哈希槽的概念，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。

## **Redis集群最大节点个数是多少？**

Redis集群预分好16384个桶(哈希槽)

## **Redis集群的主从复制模型是怎样的？**

为了使在部分节点失败或者大部分节点无法通信的情况下集群仍然可用，所以集群使用了主从复制模型,每个节点都会有N-1个复制品.

## **Redis集群会有写操作丢失吗？为什么？**

Redis并不能保证数据的强一致性，这意味这在实际中集群在特定的条件下可能会丢失写操作。

## **Redis集群之间是如何复制的？**

异步复制

## **Redis如何做内存优化？**

尽可能使用散列表（hashes），散列表（是说散列表里面存储的数少）使用的内存非常小，所以你应该尽可能的将你的数据模型抽象到一个散列表里面。比如你的web系统中有一个用户对象，不要为这个用户的名称，姓氏，邮箱，密码设置单独的key,而是应该把这个用户的所有信息存储到一张散列表里面.

##  **Redis回收进程如何工作的？**

一个客户端运行了新的命令，添加了新的数据。

Redi检查内存使用情况，如果大于maxmemory的限制, 则根据设定好的策略进行回收。

## **Redis回收使用的是什么算法？**

LRU算法

## **Redis有哪些适合的场景？**

1）Session共享(单点登录)

2）页面缓存

3）队列

4）排行榜/计数器

5）发布/订阅 

# jedis 和 redisson 有哪些区别？

# 怎么保证缓存和数据库数据的一致性？



# redis 分布式锁有什么缺陷？




