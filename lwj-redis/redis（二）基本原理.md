# redis

## 内存回收

### 过期策略

1. 定时过期

   通过redis的expire设置过期时间，每个key都需要创建一个定时器，到期删除，对内存友好。但是会占用大量cpu处理过期数据，影响缓存的响应时间和吞吐量

2. 惰性删除

   访问key时判断是否过期需要清除，节省cpu资源，对内存不友好。大量过期key未被访问导致不释放

   获取数据时候的expireIfNeed方法以及写入key发现内存不足调用activeExpireCycle释放部分内存

3. 定期过期

   间隔一定时间扫描expire字段项中的key，清除过期的key。通过调整间隔扫描时间，达到cpu和内存的最优。与定期删除的不同在于不需要实时去清除过期数据。间隔时间为redis.conf中的hz字段（单位为秒，默认为10，可配置）
   
   ![image-20200619160143595](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200619160143595.png)

- redis同时使用惰性删除以及定期过期

### 淘汰策略

**达到极限时候淘汰算法来清除数据**，最大内存通过配置文件的maxmemory参数设置，maxmemory_policy指定淘汰策略	

> LRU，Least Recently Used：最近最少使用
>
> LFU，Least Frequently Used，最不常用，根据使用频率计算，4.0 版本新增

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

   redis volatile-lru/volatile-lfu/volatile-ttl/allkeys-lru/allkeys-lfu淘汰策略原理：获取样本根据排序权值idel进行排序，从最后的元素开始淘汰，相比LRU链表成本低

   ---

   

   **redis-lru淘汰方式**：根据配置的采样值 maxmemory_samples随机获取一定数量，淘汰热度最低的值

   - 热度判断：redisObject维护了value的热度值，server.lruclock是定时任务（每100ms调用一次）更新的unix时间戳（源码：server.c#updateCachedTime）单位秒，24位只能存储194天，超过则从头计算
- 存在问题：lru采用随机取样，因此取样数量越大，淘汰数据越精确，消耗的cpu也就越高
   
   

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200611143327400.png">

---

<img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200611141102255.png">

**redis-lfu淘汰方式**：server.lruclock高16位存储时间（分钟），低8位用于存储频次（读写时更新）

根据衰减因子 lfu-decay-time（N分钟），频次会根据时间减少N

**Random**：随机删除



## 持久化机制

redis的两种持久化方案：RDB（Redis DataBase）快照、AOF（Append Only File）

**默认是RDB方式**

### RDB

![image-20200619163609121](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200619163609121.png)

内存数据写入磁盘，生成.rdb文件，重启加载该文件恢复

RDB文件解析：

1. REDIS：载入时用于判断是否是RDB文件
2. db_version：用于记录RDB版本号
3. databses：存储数据内容
4. EOF：结束标志，程序加载到这个标志代表RDB文件已经载入完毕
5. check_sum：校验标志，检验传入值和自己计算出来的值是否一致，用来判断文件是否损坏

**触发时机**：根据redis.conf配置的save参数定时备份（配置格式save   100   1）代表100秒内存在1次修改就备份

- shutdown触发，保证服务器正常关闭
- redis定时任务serverCron(每100毫秒执行一次)判断是否需要执行备份

**指令触发**：save和bgsave指令

- save指令会阻塞进程，不建议使用
- bgsave：redis进程fock子进程进行备份，fock之后的数据不写入，阻塞之间只有fock子进程的过程这段很短时间的阻塞。查看最近快照时间命令：lastsave。查看上次成功后进行的修改次数：dirty
- 执行bgsave期间执行save会被服务器拒绝：防止产生竞争条件

**优势**：

1. 适合备份和灾难恢复
2. 主进程不需要进行磁盘io操作
3. RDB恢复数据量大时效率大于AOF

**劣势**：

1. fock子线程的阻塞以及成本
2. 一定量的数据丢失：两次备份过程中shutdown导致的部分丢失

### AOF

![img](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/clipboard.png)

执行更改的redis操作指令追加到AOF文件当中，重启通过执行AOF中的指令恢复数据

**开启方式**：配置文件中appendonly   yes

> **文件增大问题**：redis重写机制，保证AOF文件大小超过指定阈值后进行内容压缩，只保留可恢复的最小指令集。
>
> 原理：读取最新数据，通过一条命令去替换过程中的命令，生成新的AOF文件
>
> 例子：
>
> 1. 初始数据SET  name  a
> 2. 过程指令APPEND name  b、APPEND name  c、APPEND name  d
> 3. 当前name的值应该是abcd   则AOF进行压缩后只会存APPEND name  bcd这条指令

**缓存写入aof文件的时机**：根据配置文件决定，默认是everysec

![image-20200619165511657](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200619165511657.png)

**重写过程中发生的写入**：存储在缓冲区当中，机制可配置

**优势**：丢失数据最多是同步频率的时间，默认1秒

**缺点**：文件大，高并发下RDB性能好

- 方案选择：最好结合使用：优先使用aof，aof处于关闭状态则使用rdb