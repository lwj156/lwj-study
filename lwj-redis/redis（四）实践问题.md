# redis

## redis性能

redis的速度有多快

每秒钟处理5万次左右的set请求

QPS可达到10万左右

**redis性能高的原因**：

1. 内存操作
2. 单线程：减少创建销毁线程的消耗，避免上下文切换的损耗，避免锁竞争
3. 多路复用：处理多个网络请求用一个redis处理线程

redis基于Reactor模式开发的网络事件处理器，也成为文件事件处理器

**什么是Reactor模式**：

1. 处理器同步等待多个事件源到达（通过linux的select方法实现）
2. 处理器将事件分发给不同的事件处理器

redis线程模型：

![image-20200621102652275](https://gitee.com/lwj156/picture/raw/master/image/redis/image-20200621102652275.png)

## redis三大问题

### 缓存穿透

查询不存在的数据导致海量请求落到数据库上

**避免方式**：

![image-20200620162107699](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200620162107699.png)

1. 缓存数据为空的数据
   > 攻击性的请求，可以让每次请求key不同，每次都缓存会导致大量数据缓存
   >
   > 缓存的数据可能一段时间后有真实数据写入，因此需要设置一定的失效时间或者更新策略

2. 布隆过滤器：根据哈希计算取模算法将数据预先存入过滤器当中，校验数据经过多次哈希计算，经过布隆过滤器判断是否存在

   > 若布隆过滤器判断不存在，则一定不存在
   >
   > 若判断存在，元素不一定存在，因为可能多个元素的hash值计算与校验数据的hash值计算相等

   **项目中使用**：Google Guava工具类中有BloomFilter实现类，可指定成功率

   ![](https://gitee.com/lwj156/picture/raw/master/image/redis/image-20200620163419323-1592642115124.png)

   

### 缓存雪崩

缓存服务器宕机，导致海量请求落到数据库上

**避免方式**：

1. 提高redis的高可用：集群
2. 达到一定失败率使用Hystrix（熔断、限流、降级）

### 缓存击穿

热点数据失效，海量请求落到数据库上

**避免方式**：

1. 互斥锁：缓存恢复过程通过互斥锁，只有一个线程用于恢复缓存，会降低系统吞吐量，若多种类型数据失效，还是会多数请求落到数据库上
2. 缓存定时预热，防止失效
3. 随机数据过期时间，尽量让失效时间离散分布
4. 永不过期：不建议

## redis应用

### 发布订阅模式

redis实现发布/订阅的方式：list消息队列、redis订阅频道（subscribe/public channel）

channel类似与queue队列的概念



### redis事务

Redis 的事务常用四个命令：multi（开启事务），exec（执行事务），discard（取消事务），watch（监视）



### Lua脚本

轻量级脚本语言，可以通过脚本执行多个命令减少开销，保证原子性

通过eval指令调用



### 分布式锁

通过setNx实现分布式锁



### redis数据加速

redis在应用中都是用来缓存热点数据，减少数据库压力，一般情况下会在第一次查询后热点数据写入缓存当中，或者提前写入缓存当中（数据预热）。客户端访问的时候，若redis中有数据则从redis中获取，否则读库写入缓存。

### 数据一致性问题

当缓存数据发生变化的时候，我们需要写入数据库，也需要更新缓存。操作情况如下：redis操作建议直接删除：若缓存数据是多表关联数据，那么还需要关联表，直接删除缓存的话更为简便，等下一次查询在写入缓存



**情况1：先操作数据库数据后删除redis的数据**

> 若先操作数据库失败，捕获异常，后续不执行，数据一致
>
> 若先操作数据库成功，redis删除失败，数据不一致：可捕获异常重试



**情况2：先删除redis数据后操作数据库的数据**

> 先redis删除失败，捕获异常，后续不执行，数据一致
>
> 先redis删除成功，后数据库更新失败，以数据库的数据为准，数据一致



并发情况下

1. 如果update接口删除了缓存，写入mysql过程前，select接口读取，发现缓存为空，读取mysql中未更新的数据，写入到了缓存中后update接口更新数据库中的数据，就会导致缓存中的未脏数据

   > 解决方式：延时双删
   >
   > 1. 先删除缓存，写入数据库，休眠一定时间后在删除缓存
   >
   >    如果没进行休眠：可能update接口更新完数据库后马上删除缓存，select接口还未写入缓存，同样会有数据不一致的问题
   >
   >    - 第二次删除缓存可能失败：可通过发送到消息队列不断重试
   >
   > 2. 设置缓存过期时间：导致一定时间的数据不一致

   ![image-20200620230122844](https://gitee.com/lwj156/picture/raw/master/image/redis/image-20200620230122844.png)

2. select接口并发写入过程：update删除缓存过程中，大量select请求判断缓存为空，导致请求落入数据库中，解决方式：

   > 乐观锁，保证只有一个线程去同步数据库数据到缓存，其他线程抢占到锁后再次判断缓存是否存在



### 热点key监控

redis-faina（https://github.com/facebookarchive/redis-faina.git）

缺点：

1. 无法监控集群节点，只能监控单节点
2. 会影响部分性能

### redis客户端

redis是通过tcp进行交互的，默认端口为6379，redis通过resp序列化协议编码数据，提高解析效率和可读性

> 项目中使用：java客户端主要有 Jedis，Redisson 和 Luttuce

**jedis**：

> 请求模式：client（单指令）、pipeline（批处理）、事务

1. 非线程安全：共享socket 

**Lettuce**：

> 基于netty架构实现，redisTemplate底层默认使用Lettuce

**redission**：

> 基于netty实现，采用非阻塞IO，性能高，不支持事务
>
> 基于redis实现的分布式服务，可实现分布式事务

### redis优化策略

存入redis中的数据需要有闭环的思路，数据何时产生，何时销毁，何时更新

**数据如果没有删除的api，尽量设置过期时间，防止业务随时间推进后的redis堆积**

开发过程中的优化方式：

1. 批处理语句可以用pipeline，减少网络往返时间和IO读写次数
2. 使用hash结构的时候考虑下，以后的业务是否需要针对field做过期，如果需要，建议使用String结构
3. set集合取交并集合的时候无法跨节点，需要注意
4. 需要考虑主从同步的延迟时间会不会导致业务问题
5. 定期根据快照文件排查redis各个key的状态，推荐可视化工具：github的雪球rar
6. 优化慢查询日志：slow-log-log-slower-than配置慢查询时间界限（单位微秒）、SLOWLOG GET获取慢查询日志

