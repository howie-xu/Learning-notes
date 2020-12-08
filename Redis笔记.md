# Redis笔记

###  Redis由来

+ 08年    意大利antirez    redis之父
+ 需求原型：统计每个人对每个站点的访问情况
+ 09年的时候，antirez发明了内存数据库redis，C语言
+ 全称 Remote Dictionary Service，远程字典服务
+ 默认端口6379，代表他仇恨的一个意大利广告女明星 MERZ（数字对应的手机键盘字母），MERZ代表愚蠢

### Redis特性

+ 单线程    

   命令执行的时候是单线程

+ 多路复用
+ KV   v可以有很多类型，k最大大小为512M（String类型最大大小）
+ 持久化   过期策略   淘汰策略
+ 高可用   集群

### Redis常见数据类型以及应用场景

+ Redis有几大数据类型？

   八种，常用的有五种（strings、hashes、lists、sets、sorted sets、bitmaps、hyperloglogs、geospatial indexes）

#### string

get    set

使用场景：

1. 唯一ID、计数、软限流(访问量达到一定数量进行限流)

   incr key1：自增1  \  incr key1 100：自增100

2. 分布式锁

   setnx   expire

3. 缓存

   对于对象中的数据存储，可以将对象序列化为Json字符串存储。但是当需要对对象中的某个属性进行修改时会比较麻烦，这个时候就需要使用hash类型

#### hash

hash的value是key-value对

命令：hget   hset    hmset   hmget   hkeys   hincrby   hdel    hdel   hlen

使用场景：

1. 购物车

   模型—   key: 用户id     field: 商品id      value: 商品数量

   操作—  +1：hincrby hkey f  1

   ​              -1:   hincrby hkey f  -1

   ​              删除： hdel  f

   ​              全选： hgetall

   ​              商品数：hlen

2. string能做的hash原则上都能做

#### list

有序、可以重复

lpush   rpush 

lrange  rrange

lpop  rpop

blpop  brpop(阻塞队列)，对应block的命令，redis有单独处理，不会因为单线程执行命令而阻塞其他命令；在等待的时候，如果有其他客户端把数据放进来，会弹出去

使用场景：

1. 消息队列（不要实现这个，因为队列有成熟的实现中间件mq，并且redis的队列发出去的消息无法响应，即无法进行ack）
2. 有序的列表、时间线列表

#### set

无序、不可重复

sadd   smembers   scard（大小）    spop(随机弹)   srem(指定弹元素)    srandmember(随机获取元素) sismember(是否存在元素)    sdiff(前面一个有但是后边一个没有的元素)    sinter(交集)    sunion(并集)

使用场景：

1. 共同关注、可能认识（sinter、sdiff）
2. 抽奖（spop、srandmember）

#### zset

有序、不可重复

根据什么有序？score，如果key不同，但是score相同，根据key的ASCII码进行排序

zadd    zrem(删除元素)   zrange(score从低到高获取)    zrevrange(score从高到低获取)   zrangebyscore(获取分数范围内元素)   zincrby(给某一个元素加分)  zscore(获取元素的分数)    zrank(获取分数排名)

使用场景：

1. 排行榜

### Lua脚本

胶水语言，不能独立运行，必须依赖宿主

Redis支持Lua语言

一次性发送多个命令，并且保证多个命令原子性，redission底层就是用lua语言实现

为什么要使用lua？举例：实现分布式锁时，需要先setnx加锁，然后expire设置过期时间，但是两条命令无法保证原子性，当加锁命令执行完后线程挂掉，则未添加过期时间导致锁无法被释放

~~~lua
eval script numkeys key [key ...] arg [arg ...]
eval "redis.call('set', KEYS[1], ARGV[1]) return KEYS[1]" 1 k1 v1
eval "redis.call('setnx', KEYS[1], ARGV[1]) if lockSet == 1 then redis.call('expire', KEYS[1], ARGV[2]) end renturn lockSet" 1 k1 v1 time
~~~

### 常用命令

Redis默认16个库，可以在redis.conf文件中配置

info replication：查询节点信息

select  xxx(1): 用于切换数据库

dbsize 查询库

flushall：清除所有库中数据

flushdb：清除当前库中数据

keys *：查询所有key，可能会将机器卡死

rename key1 key2: 修改key名

expire key1 10：对key1设置过期时间为10s

ttl key1: 查询key1过期时间

exist key1：查询key1是否存在

type key1：查询key1对应值的类型

mset key1 value1 key2 value2：批量设置

mget key1 key2：批量获取

strlen key1：获取value长度

append key1 tmp：

incr key1：自增1（场景：用作全局自增id、浏览量点击量 等）incr key1 100：自增100

setnx k1 v1 (set if not exists，常用作分布式锁)

### Redis作为内存数据库带来的问题

redis.conf 配置文件

maxmemory 最大得内存，默认为机器最大内存

问题：内存耗尽

解决思路：1. 过期数据，过期策略   2. 未过期，淘汰策略

### Redis的过期策略

针对设置了过期时间的key，如果过期则进行删除

#### 被动（惰性）过期

只有访问的时候，才去判断这个数据是否过期

对一个key设置过期时间为10s，只要不去访问数据，则该数据一直存在内存中

缺点：对内存不友好

优点：对CPU友好

#### 定期过期

1. 定期周期

   serverCron 时间事件驱动的概念 ，redis定期被调用的方法，执行定期过期、定期关闭无用客户端连接、更新服务器统计数据、尝试持久化操作、获取当前时间等操作

   hz参数决定定期周期，默认为hz 10（即1s执行10次，100ms一次）

2. 周期扫描范围

   设置了过期时间的key中扫描

3. 怎么去过期（6.0.8版本）

   + 链地址法，在hash桶中获取设置了过期时间的key。每个hash桶数据取完，但是如果拿到的key超过20个，则不继续对下一个hash取值。如下：

     假如第一个hash桶有25个， 则获取到25个key，并且不继续对下一个桶操作；

     假如第一个hash桶有10，第二个hash桶15，则获取25

     最多能拿400个hash桶

   + 删除拿出来的数据中过期的key

   + 假如拿到的数据过期比例超过10%，或者400个桶都没值，则会重复前两步

   + 循环16次后，有时间限制（防止死循环或者时间占用太久）
   
   ![redis定期过期策略实现](https://imgs-seven.vercel.app/redis/Redis-Exp.png)
### Redis的淘汰策略

| 含义                                                         | 策略            |
| ------------------------------------------------------------ | --------------- |
| 根据LRU算法删除设置了超时属性（expire）的键，直到腾出足够的内存为止。如果没有可删除的键对象，回退到noeviction策略。 | volatile-lru    |
| 根据LRU算法删除键，不管数据有没有设置超时属性，直到腾出足够的内存为止。 | allkeys-lru     |
| 在带有过期时间的键中选择最不常用的。                         | volatile-lfu    |
| 在所有的键中选择最不常用的。                                 | allkeys-lfu     |
| 在带有过期时间的键中随机选择。                               | volatile-random |
| 在所有的键中随机选择。                                       | allkeys-random  |
| 根据键值对象的ttl属性，删除最近即将要过期数据。如果没有，回退到noeviction策略。 | volatile-ttl    |
| 默认策略，不会删除任何数据，拒绝所有写入操作并返回客户端错误信息（error）OOM commond not allowed when used memory，此时Redis只响应读操作。 | noeviction      |

~~~c
typedef struct redisObject {
unsigned type:4;
unsigned encoding:4;
unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
* LFU data (least significant 8 bits frequency
* and most significant 16 bits decreas time). */
int refcount;
void *ptr;
} robj;
~~~

#### LRU

Least Recently Used，最久未使用，衡量标准为时间

redis数据对象redisObject中lru参数，24bit，保存的是以秒为单位的该对象上一次被访问的时间 

每次数据对象操作被访问的时候，redisObject.lru=当前时间的秒单位的最后24位

~~~java
101011101111111111111111001010
    & 111111111111111111111111
    --------------------------
      101111111111111111001010
# 当前时间秒单位 & (2^24-1)二进制，得到二进制后24位
~~~

![redis定期淘汰策略实现](https://imgs-seven.vercel.app/redis/Redis-LRU.png)

#### LFU

Least Frequently Used，最不常用，衡量标准为访问次数

redis数据对象redisObject中lru参数，24bit，高16位以“分”为单位记录上一次访问redisObject的时间，低8位按照某种特定的算法计算过去的某段时间间隔内该对象的使用频次 

创建时：

redisObject.lru = 当前时间的分单位的最后16位(高16位)  +  00000101(低8位，默认counter=5)
修改时：

redisObject.lru = 当前时间的分单位的最后16位(高16位)  +  counter(低8位)

![redis定期淘汰策略实现](https://imgs-seven.vercel.app/redis/Redis-LFU.png)

 当在近期内使用了某个键的时候，那么要对该键进行增加计数，但是只是执行增加计数的操作，实际能不能完成增加计数，这种情况是随机概率的且这种概率随着当前使用频次的增大而减小。可能性随着计数值的增大呈现出如下变化规律：1.0/((counter-LFU_INIT_VAL) * server.lfu_log_factor+1)，如图所示。当达到计数能够表示的最大值255的时候，直接返回该计数。 

![redis定期淘汰策略实现](https://imgs-seven.vercel.app/redis/LFU-计数增加概率分布图.png)

### Redis持久化策略

#### rdb 

Redis database ，默认的持久化，生成快照文件

redis.conf配置：dbfilename "dump.rdb" 当前快照文件名，dir "/usr/local/redis-5.0.5/scr" 快照文件路径

1、自动触发

+ 满足save配置：

​       save 900 1 --->900秒内有1个key被修改了 

​       save 300 10 

​       save 60 10000

​       多条配置满足任意一条即可触发rdb

+ shutdown命令

+ flushall命令

2、手动触发

- SAVE ，直接调用 rdbSave ，阻塞 Redis 主进程，直到保存完成为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
- BGSAVE，fork 出一个子进程，子进程负责调用 rdbSave ，并在保存完成之后向主进程发送信号，通知保存已完成。 Redis 服务器在BGSAVE 执行期间仍然可以继续处理客户端的请求。

#### aof（推荐）

Append only file，默认不开启，将redis命令记录到文件

redis.conf配置：appendonly no默认不开启，appendfilename "appendonly.aof" 命令记录文件

两个都开启的话，以aof作为持久化策略

##### 重写机制：

优化命令记录文件大小和恢复速率

重写触发条件：

+ auto-aof-rewrite-percentage 100  日志大小比例扩容100%（上次重写后日志文件100M，当日志文件达到200M时进行重写）
+ auto-aof-rewrite-min-size 64mb  日志大小达到64M，进行重写

重写效果：lpush l1 v1; lpush l1 v2 v3; lpop l1;   ----> lpush l1 v1 v2 

### Redis哨兵sentinel

集群  主从   高可用   负载

![redis定期淘汰策略实现](https://imgs-seven.vercel.app/redis/sentinel.jpg)

配置文件sentinel.conf 

~~~yaml
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
#
# Tells Sentinel to monitor this master, and to consider it in O_DOWN
# (Objectively Down) state only if at least <quorum> sentinels agree.
#
# Note that whatever is the ODOWN quorum, a Sentinel will require to
# be elected by the majority of the known Sentinels in order to
# start a failover, so no failover can be performed in minority.
#
# Replicas are auto-discovered, so you don't need to specify replicas in
# any way. Sentinel itself will rewrite this configuration file adding
# the replicas using additional configuration options.
# Also note that the configuration file is rewritten when a
# replica is promoted to master.
#
# Note: master name should not include special characters or spaces.
# The valid charset is A-z 0-9 and the three characters ".-_".
sentinel monitor mymaster 127.0.0.1 6379 2


daemonize yes
port 26379
protected-mode no
dir “/usr/local/redis-5.0.5/sentinel-tmp”

sentinel down-after-milliseconds redis-master 30000
sentinel failover-timeout redis-master 180000
sentinel parallel-syncs redis-master 1
~~~



#### sentinel之间如何通信

sentinel只配置了一个主节点的地址和主节点主机名

+ ping
+ info replication
+ publish 订阅发布   HELLO频道

#### 如何判断主节点下线

+ 主观下线

  sentinel A  ping master节点，如果一定时间内（down-after-millseconds）内没有回复，sentinel A就会觉得master节点要下线了

+ 客观下线

  sentinel  A询问其他sentinel ，其他的sentinel 觉得也下线，则master真正下线

#### 哪个sentinel去执行提升从节点为主节点

选举  raft算法  少数服从多数  超过50%，先到先得

sentinel需要为奇数个，不然可能无法形成多数派，会产生脑裂

#### 升哪个从节点为主节点

断开连接时长  优先级队列（可以配置）  从主节点复制数量   进程ID

### Redis缓存问题

使用建议：

+ 接口缓存（要求性能高，用户相关的不能使用）、内容缓存
+ 数据修改时，建议对redis中key进行删除，而不是修改
+ 删除操作先操作reids，再操作DB；其他情况先操作DB，再操作redis；
+ 

#### 缓存一致性

![redis定期淘汰策略实现](https://imgs-seven.vercel.app/redis/缓存一致性问题.jpg)

#### 缓存穿透

![redis定期淘汰策略实现](https://imgs-seven.vercel.app/redis/缓存穿透.jpg)

#### 缓存雪崩

### 相关源码

+ redisDb

~~~c
typedef struct redisDb {
dict *dict; /* The keyspace for this DB */
dict *expires; /* Timeout of keys with a timeout set */
dict *blocking_keys; /* Keys with clients waiting for data (BLPOP)*/
dict *ready_keys; /* Blocked keys that received a PUSH */
dict *watched_keys; /* WATCHED keys for MULTI/EXEC CAS */
int id; /* Database ID */
long long avg_ttl; /* Average TTL, just for stats */
} redisDb;
~~~

重点关注的其中的三个变量dict *dict、dict *expire以及id。

id代表了该db在redisServer中的编号。redisServer一共存在dbnum个redisDb，server.dbnum是在redis启动的时候设置的，通过server.dbnum=CONFIG_DEFAULT_DBNUM(默认是16)进行初始化。dict用于存放所有的键值对，无论是否设置了过期时间，expire只用于存放设置了过期时间的键值对的值对象。 

+  redisObject 

~~~c
typedef struct redisObject {
unsigned type:4;
unsigned encoding:4;
unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
* LFU data (least significant 8 bits frequency
* and most significant 16 bits decreas time). */
int refcount;
void *ptr;
} robj;
~~~

type代表了redis中的键对应的值是redis提供的五种对象之中的哪种类型;encoding是这种对象的底层实现方式，redis为每种对象至少提供了两种实现方式;lru，在使用lru相关的策略时，其24位保存的是以秒为单位的该对象上一次被访问的时间，如果使用的是lfu，那么高的16位以分为单位保存着上一次访问的时间，低8位保存着按照某种规则实现的某段时间内的使用频次。以上元素都是使用“位域”实现，节省内存。refcount保存着该对象的引用计数，用于对象的共享和释放。ptr实际指向了该对象的底层实现。 

+ estimateObjectIdleTime(LRU中计算key多久时间未访问算法)

~~~c
unsigned long long estimateObjectIdleTime(robj *o) {
unsigned long long lruclock = LRU_CLOCK();
/* if条件代表了lruclock和o->lru都在同一个范围之内的情况，表示没有发生回绕 */
if (lruclock >= o->lru) {
return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
/* 时钟的大小超过了lruclock能够表示的范围，发生了回绕，因此间隔的时间等于lruclock加上LRU_CLOCK_MAX减去o->lru。
* linux内核中的jiffies回绕解决方案设计的更为精妙，可以参考《linux内核设计与实现》一书或博客（此处为超链接）的讲解。
*/
} else {
return (lruclock + (LRU_CLOCK_MAX - o->lru)) *
LRU_CLOCK_RESOLUTION;
}
}
~~~

+ LFUDecrAndReturn（LFU中根据未访问时长计算counter算法）

~~~c
/* If the object decrement time is reached decrement the LFU counter but
 * do not update LFU fields of the object, we update the access time
 * and counter in an explicit way when the object is really accessed.
 * And we will times halve the counter according to the times of
 * elapsed time than server.lfu_decay_time.
 * Return the object frequency counter.
 *
 * This function is used in order to scan the dataset for the best object
 * to fit: as we check for the candidate, we incrementally decrement the
 * counter of the scanned objects if needed. */
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;
    unsigned long counter = o->lru & 255;
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;
    return counter;
}
~~~

+ LFULogIncr（LFU中key被访问后增加counter算法）

~~~c
/* Logarithmically increment a counter. The greater is the current counter value
 * the less likely is that it gets really implemented. Saturate it at 255. */
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    double r = (double)rand()/RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    if (r < p) counter++;
    return counter;
}
~~~

