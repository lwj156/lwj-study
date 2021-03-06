# redis

> 后续源码文件名称统一通过()表示，redis底层是C语言，因此.h、.c文件可认为是源码文件
>
> 源码版本 redis-6.0.5

redis全称：REmote DIctionary Service   译为远程字典服务

每个KV键值对都存储在dictEntry(dict.h)里面，redis底层是哈希表（hashTable），结构体现在dictEntry是数组，*next指针维护链表。

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200617173236796.png">

key是字符串结构，redis采用sds结构（sds.h）存储，由于底层是C语言编写，C语言使用的字符串格式无法满足redis对于字符串的要求。

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618140613382.png">

**sds与char[]的区别**：

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618141005606.png">

1. sds长度存储在len属性当中，无需再计算
2. 内存分配不足导致溢出，sds会提前检查空间剩余量并进行分配
3. 减少内存分配次数，sds修改后len变为10字节，而buf数组实际长度为21，10字节空余，1字节用于存储空字符串
4. 二进制安全问题：C语言通过空字符串识别结束，像视频、图片、音频等中间内容包括了空字符，会导致解析失败

value存储在redisObject当中（server.h）

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618134749915.png">

对象类型可以根据命令：type key查看

## 基本数据结构

### String

**基本介绍**：

字符串类型的内部编码（对应redisObject中encoding存储的值）

可通过命令object encoding key查询编码

1. int：（8字节  64位）
2. embstr：sds的一种格式，存储小于44字节的字符串 ，redisObject和sds连续分配，修改时需要重新全部分配，因此设置为只读
3. raw：存储大于44字节的字符串，redisObject和sds分别分配。

embstr以及raw的临界值可通过配置#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44实现（object.h）超过范围则自动转换类型，修改数据会转换成raw类型，大内存编码无法退化为小内存编码。

**应用场景**：

1. 热点内容缓存
2. 分布式session
3. 分布式锁：setnx
4. 全局id：incrby    项目上应用：全局msgId
5. 原子性操作可应用的场景：限流、计数

### Hash

value只能是字符串类型

**基本介绍**：

1. 优点：减少内存空间、减少key冲突、批量获取减少IO
2. 缺点：field无法单独设置过期时间、无bit操作、数据量分布问题（都分布在key所在节点）
3. 基本操作指令：hset、hmset、hscan

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200609133918596.png">

**实现原理：（底层由两种数据结构组成）**

hash的底层由两种结构组成：ziplist和hashtable

1. ziplist压缩列表（ziplist.c）：

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618174939538.png">

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618175323429.png">

   通过当前节点的长度和上一个节点的长度计算出上一个节点的地址

   使用场景：hash键值都小于64byte、键值对数量小于512个

   可通过redis.conf配置转成哈希表的临界值：hash-max-ziplist-value、hash-max-ziplist-entries

   当超过这两个阈值的时候转换为hashtable（哈希表）

   **编码格式**：根据字节区分

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618175446701.png">

2. hashtable哈希表（dict.h）：redis底层哈希表可参照此结构

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200609155406544.png">

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618175859237.png">

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618180044562.png">

   根据负载因子就是链表的长度（dict_force_resize_ratio=5），若大于，则触发rehash，扩容大小为当前库大小*2

   **rehash过程**： redis-rehash过程是采用渐进式rehash即分多次、渐进式完成的

   1. 扩容大小h[1]为h[0]使用的数量*2，讲rehashidex索引计数器置0，开始rehash
   2. 程序进行crud的过程中外，还迁移rehashidex上的节点，重新计算hash值，迁移h[0]节点到h[1]当中，rehashidx+1
   3. rud操作（即除了新增操作）会两个哈希表同时操作，新增操作只操作h[1]表，保证h[0]中的数据只减不增，最后变成空表释放空间

   **应用场景**：

   1. 用户的任务进度：key-用户id    field：任务名称   value：任务进度
   2. 用户的购物车：key-用户id   field：商品   value：商品数量

### List

左右都可取出数据

**实现原理**：

3.2版本之后采用quicklist来存储

![image-20200618190010964](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618190010964.png)

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200609183912407.png">



<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200609183927880.png">

**应用场景**：

1. 由于list是有序排列，因此可以作为时间线
2. 消息队列：BLPOP/BRPOP指令，阻塞等待，有数据才弹出

### Set

**基本介绍**：无序集合（最大存储40亿数据）

**实现原理**：根据数据类型用不同的数据结构

1. 整数类型：inset存储
2. 非整数类型：hashtable

**应用场景**：

1. 漂流瓶等从集合随机抽取的活动
2. 点赞、签到、喜欢的用户等：key-like+uid   value-对象id
3. 商品标签：通过交集并集

### Zset

**基本介绍**：有序集合

**实现原理**：源码结构（server.h）

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618191330929.png">

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200618190932407.png">

1. 元素数量小于128，长度小于64字节用ziplist
2. 超过阈值后，用skiplist+dict存储（跳跃表）:跳跃表就是维护了多层数据，根据当前层判断是否需要到下层查找

**应用场景**：key  score field

1. score为时间戳，可以统计指定时间段内的数据
2. zrevrange：获取排名

### Bitmaps

**基本介绍**：位运算，8位2进制组成

常用指令：bitcount   k1  统计二进制位中1的个数

**应用场景**：

1. 用户访问统计：位运算
2. 在线用户统计：维护串很长的二进制，用户根据id，在线就修改指定位置为1，统计可通过bitcount



### Hyperloglogs

**基本介绍**：位运算，8位2进制组成

**基本原理**：与布隆过滤器有类似作用

**应用场景**：海量数据统计问题



### Streams

**基本介绍**：原理类似kafka

**应用场景**：可实现发布订阅功能



## 基本应用

## 内存回收

### 过期策略

1. 定时过期

   通过redis的expire设置过期时间，每个key都需要创建一个定时器，到期删除，对内存友好。但是会占用大量cpu处理过期数据，影响缓存的响应时间和吞吐量

2. 惰性删除

   访问key时判断是否过期需要清除，节省cpu资源，对内存不友好。大量过期key未被访问导致不释放

   获取数据时候的expireIfNeed方法以及写入key发现内存不足调用activeExpireCycle释放部分内存

3. 定期过期

   间隔一定时间扫描expire字段项中的key，清除过期的key。通过调整间隔扫描时间，达到cpu和内存的最优。与定期删除的不同在于不需要实时去清除过期数据

- redis同时使用惰性删除以及定期过期

### 淘汰策略

内存达到极限时候淘汰算法来清除数据，最大内存通过配置文件的maxmemory参数设置，maxmemory_policy指定淘汰策略

LRU，Least Recently Used：最近最少使用

LFU，Least Frequently Used，最不常用，根据使用频率计算，4.0 版本新增

1. 淘汰策略

   | 淘汰策略        | 说明                               |
   | --------------- | ---------------------------------- |
   | volatile-lru    | 设置expire数据中使用次数最少       |
   | allkeys-lru     | 所有数据中使用次数最少             |
   | volatile-ttl    | 设置expire数据中淘汰即将过期的数据 |
   | volatile-random | 设置expire数据中随机淘汰           |
   | allkeys-random  | 所有数据随机淘汰                   |
   | no-enviction    | 所有写入操作报错，读操作正常       |
   | volatile-lfu    | 设置expire数据中选择最不常用的     |
   | allkeys-lfu     | 在所有的键中选择最不常用的         |

   **传统LRU算法内部原理**：通过链表维护缓存，最新缓存放在头部。如果缓存被访问，则迁移到链表头部。淘汰数据从尾部开始淘汰即可。淘汰对象追加到AOF文件当中

   

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200611113633207.png">

   redis volatile-lru/volatile-lfu/volatile-ttl/allkeys-lru/allkeys-lfu淘汰策略原理：获取样本根据排序权值idel进行排序，从最后的元素开始淘汰，相比LRU链表成本低。

   ---

   

   **redis-lru淘汰方式**：根据配置的采样值 maxmemory_samples随机获取一定数量，淘汰热度最低的值

   - 热度判断：redisObject维护了value的热度值，server.lruclock是定时任务更新的unix时间戳（源码：server.c）单位秒，24位只能存储194天，超过则从头计算

   

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200611143327400.png">

---

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200611141102255.png">

**redis-lfu淘汰方式**：server.lruclock高16位存储时间（分钟），低8位用于存储频次（读写时更新）

根据衰减因子 lfu-decay-time（分钟），频次会根据时间减少

   **Random**：随机删除

## 持久化机制

redis的两种持久化方案：RDB（Redis DataBase）快照、AOF（Append Only File）

1. RDB：内存数据写入磁盘，生成.rdb文件，重启加载该文件恢复

   **触发时机**：根据redis.conf配置的save参数定时备份

   - shutdown触发，保证服务器正常关闭

   **指令触发**：save和bgsave指令

   - save指令会阻塞进程，不建议使用
   - bgsave：redis进程fock子进程进行备份，fock之后的数据不写入，阻塞之间只有fock子进程的过程这段很短时间的阻塞。查看最近快照时间命令：lastsave

   **优势**：

   1. 适合备份和灾难恢复
   2. 主进程不需要进行磁盘io操作
   3. RDB恢复数据量大时效率大于AOF

   **劣势**：

   1. fock子线程的阻塞以及成本
   2. 一定量的数据丢失：两次备份过程中shutdown导致的部分丢失

2. AOF：执行更改的redis操作指令追加到AOF文件当中，重启通过执行AOF中的指令恢复数据

   > **文件增大问题**：redis重写机制，保证AOF文件大小超过指定阈值后进行内容压缩，只保留可恢复的最小指令集。
   >
   > 原理：读取最新数据，通过一条命令去替换过程中的命令，生成新的AOF文件
   >
   > 例子：
   >
   > 1. 初始数据SET  name  a
   > 2. 过程指令APPEND name  b、APPEND name  c、APPEND name  d
   > 3. 当前name的值应该是abcd   则AOF进行压缩后只会存APPEND name  bcd这条指令

   **优势**：丢失数据最多是同步频率的时间，默认1秒

   **缺点**：文件大，高并发下RDB性能好

3. 方案选择：最好结合使用

## Redis集群

保证redis的性能、扩展以及可用性。
