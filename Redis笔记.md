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

   + 在hash桶中获取设置了过期时间的key。每个hash桶数据取完，但是如果拿到的key超过20个，则不继续对下一个hash取值。如下：

     假如第一个hash桶有25个， 则获取到25个key，并且不继续对下一个桶操作；

     假如第一个hash桶有10，第二个hash桶15，则获取25

     最多能拿400个hash桶

   + 删除拿出来的数据中过期的key

   + 假如拿到的数据过期比例超过10%，或者400个桶都没值，则会重复前两步

   + 循环16次后，有时间限制（防止死循环或者时间占用太久）
   
   ![redis定期过期策略实现](.\redis定期过期策略实现.png)

### Redis的淘汰策略

| 策略            | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| volatile-lru    | 根据LRU算法删除设置了超时属性（expire）的键，直到腾出足够的内存为止。如果没有可删除的键对象，回退到noeviction策略。 |
| allkeys-lru     | 根据LRU算法删除键，不管数据有没有设置超时属性，直到腾出足够的内存为止。 |
| volatile-lfu    | 在带有过期时间的键中选择最不常用的。                         |
| allkeys-lfu     | 在所有的键中选择最不常用的。                                 |
| volatile-random | 在带有过期时间的键中随机选择。                               |
| allkeys-random  | 在所有的键中随机选择。                                       |
| volatile-ttl    | 根据键值对象的ttl属性，删除最近即将要过期数据。如果没有，回退到noeviction策略。 |
| noeviction      | 默认策略，不会删除任何数据，拒绝所有写入操作并返回客户端错误信息（error）OOM commond not allowed when used memory，此时Redis只响应读操作。 |

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

![redis定期过期策略实现](.\Redis LRU算法实现.png)

#### LFU

Least Frequently Used，最不常用，衡量标准为访问次数

redis数据对象redisObject中lru参数，24bit，高16位以“分”为单位记录上一次访问redisObject的时间，低8位按照某种特定的算法计算过去的某段时间间隔内该对象的使用频次 

创建时：

redisObject.lru = 当前时间的分单位的最后16位(高16位)  +  00000101(低8位，默认counter=5)

高16位=当前时间的分单位的最后16位