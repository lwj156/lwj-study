# redis

redis全称：REmote DIctionary Service   译为远程字典服务

![image-20200610095302423](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200610095302423.png)



## 基本数据结构

### String

### hash：value只能是字符串类型

#### 基本介绍

1. 优点：减少内存空间、减少key冲突、批量获取减少IO
2. 缺点：field无法单独设置过期时间、无bit操作、数据量分布问题（都分布在key所在节点）
3. 基本操作指令：hset、hmset、hscan

<img src="C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200609133918596.png" alt="image-20200609133918596" style="zoom:80%;" />

#### 实现原理（底层由两种数据结构组成）

1. ziplist（压缩列表）：![image-20200609152857683](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200609152857683.png)

   通过当前节点的长度和上一个节点的长度计算出上一个节点的地址

   使用场景：hash键值都小于64byte、键值对数量小于512个

   可通过redis.conf配置：hash-max-ziplist-value、hash-max-ziplist-entries

   当超过这两个阈值的时候转换为hashtable（哈希表）

2. hashtable（哈希表）：![image-20200609155406544](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200609155406544.png)

   根据负载因子（dict_force_resize_ratio=5），若大于，则触发rehash，扩容大小为当前库大小*2

### List：左右都可取出数据

#### 实现原理

3.2版本之后采用quicklist来存储

![image-20200609183912407](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200609183912407.png)

![image-20200609183927880](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200609183927880.png)

![image-20200609184016745](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200609184016745.png)

#### 应用场景：由于list是有序排列，因此可以作为时间线

### set：无序集合（最大存储40亿数据）

### zset：有序集合

#### 实现原理

1. 元素数量小于128，长度小于64字节用ziplist
2. 超过阈值后，用skiplist+dict存储（跳跃表）

#### 应用场景：key  score field

1. score为时间戳，可以统计指定时间段内的数据
2. zrank：获取排名

### Bitmaps：位运算

一个字节由8个二进制位组成

bitcount   k1  统计二进制位中1的个数

#### 应用场景

1. 用户访问统计：
2. 在线用户统计：维护串很长的二进制，用户根据id，在线就修改指定位置为1，统计可通过bitcount

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

   ![image-20200611113633207](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200611113633207.png)

   redis volatile-lru/volatile-lfu/volatile-ttl/allkeys-lru/allkeys-lfu淘汰策略原理：获取样本根据排序权值idel进行排序，从最后的元素开始淘汰，相比LRU链表成本低。

   ---

   

   **redis-lru淘汰方式**：根据配置的采样值 maxmemory_samples随机获取一定数量，淘汰热度最低的值

   - 热度判断：redisObject维护了value的热度值，server.lruclock是定时任务更新的unix时间戳（源码：server.c）单位秒，24位只能存储194天，超过则从头计算

     ![image-20200611143327400](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200611143327400.png)

   ![image-20200611141102255](C:\Users\LWJ\AppData\Roaming\Typora\typora-user-images\image-20200611141102255.png)

   ---

   

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

保证redis的性能、扩展以及可用性