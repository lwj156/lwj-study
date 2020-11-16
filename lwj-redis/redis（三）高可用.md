# redis

## 高可用方案

### 	主从复制（replication）

从节点无法写入数据，只能读取数据

**配置主从方式**：从节点配置文件中添加slave  of  ip:port

**实现原理**：从节点设置了主节点后，通过replication.c#replicationCron方法与master节点建立socket连接，每隔1秒执行一次。连接成功后从节点会创建一个专门处理复制工作的事件处理器，用于命令传播以及接收RDB文件等。

**全量数据同步过程**：

![](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200620101811594.png)

1. 第一次连接同步，slave向master发送PSYNC（2.8版本后）指令，master node通过bgsave全量复制一份RDB文件传输给slave node，过程超过一定时间会进行超时重试（可配置repl-timeout参数）
2. slave node收到RDB文件后，会先清除自身的旧的数据，在通过RDB文件加载数据
3. 同步过程中产生的指令会缓存在内存中，等待RDB文件写入slave node后在追加指令
4. 同步结束后，master node会持续将写命令传输给slave node节点

- 如果磁盘性能低，可通过无磁盘化传输RDB文件，即在内存中创建传输

  （配置参数：repl-diskless-sync yes）

**增量数据同步过程**：与全量不同的是，slave发送的psync指令会带有RUNID（服务器运行的ID）以及OFFSET（偏移量），master节点比对RUNID（第一次同步没有此数据）后进行增量同步。

- 需要注意，若增量同步数据大于复制积压缓冲区大小，则会进行全量同步

  复制缓冲区（replication backlog）：缓存一定时间的写指令，默认大小为1M

- 注意区分输入输出缓冲区：该缓冲区是用于客户端I/O交互的缓冲区

**命令传输过程的延迟**：写命令同步给slave节点过程中会有数据不一致的现象

可通过配置repl-disable-tcp-nodelay no

yes：对TCP数据进行包合并（可理解为批量传输）减少发送频率，默认为40ms

no：实时传输，master的写指令马上传输到slave节点，频率和带宽会变高

**缺点**：

1. RDB同步时间耗时
2. 单单主从部署会有可用性问题，master节点挂掉需要手动重启

### 哨兵机制（sentinel）

哨兵节点：特殊的状态的redis服务

客户端通过连接sentinel节点，可获取到当前的master节点信息

哨兵尽量为奇数数量，防止脑裂问题（多个leader节点）

![image-20200620111009540](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200620111009540.png)



**监控原理**：master节点一定时间没发送心跳请求给sentinel节点的话，会被标记下线，选举新的slave节点为master节点

**标记下线的过程（故障转移）failover**：

1. 标记下线（主观下线）：某个sentinel发现master节点超过时间（down-after-miliseconds）没发送心跳请求，该节点被标记为主观下线
2. 询问其他监控节点的判定结果：多数认为下线则标记该节点为客观下线
3. 通过leader选举出执行操作的sentinel节点（raft算法）
   - 先发现故障节点的sentinel开始投票
   - 超过半数票的节点作为leader
4. sentinel  leader节点通过slave no one和slave of指令重新指定master节点
5. 原先下线的master节点恢复后会变成slave节点

**选举新的master节点的指标**：

1. 断开连接时长、优先级排序（replica-priority 1）、复制数量（从master同步数据的增量复制数）、进程id（越小越优先）
2. 断开时间过长可能丢失选举权

**sentinel的几个超时指标**：

1. failover_start_time：当此选举失败后，与下一次选举之间的间隔时间。默认是当前时间加上1s内的随机毫秒数
2. failover_state_change_time：故障转移中标记节点状态变更的时间
3. failover_timeout：故障转移过程的超时时间。默认是3分钟
4. election_timeout：sentinel选举超时时间，是默认选举超时时间和failover_timeout的最小值。默认是10s

**缺点**：

1. 主从切换过程中的数据丢失：完整数据保存在master当中，同步给slave过程中，master节点宕机，可能导致slave不是完整数据，slave升级成新的master节点，会导致数据丢失
2. 写节点无法水平扩容，哨兵机制适用在一主多从的部署当中，数据无法分片，分片需要部署多组主从

## 分布式架构方案

### redis分片思路

**客户端路由**：客户端可以通过取模、类似cluster模式、一致性hash的方式进行分片，在分局key计算路由到哪个机器上面。仿照cluster方式可以将槽位与机器的关系缓存到本地中，如（1，100）路由到第一台机器当中

- jedis提供了客户端分片的api，可自定义分片规则

**代理服务路由**：如Twemproxy和codis，原理类似cluester模式：redis节点分别管理一定的槽位，key值根据CRC32计算后模上槽位的个数，余数对应到槽位对应的节点上，有对应的扩容机制。

缺点：

1. 需要保证proxy的可用性，需要引入zk协调
2. 部分命令不支持

![image-20200620133426732](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200620133426732.png)

### redis cluster：服务端分片策略

客户端不需要关心数据分布在哪个节点上：redis节点会自行转发

**数据分布方式**：

1. hash取模根据余数分布到不同redis节点

2. 一致性哈希：总结为哈希环结构，数据归属顺时针方向的redis节点，图中data落入redisA-B之间，顺时针方向就属于redisB，因此data数据落在redisB当中

   <img src="https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200620135548099.png" width=60%>

3. 虚拟槽（redis使用的分布策略）：

计算方式：key进行CRC16计算后%16384得到的slot值，数据落入负责对应范围slot的redis节点当中

槽位的数量不会改变，都是16384个槽位，redis分片规则以及扩容机器可配置修改。

**修改方式**：

添加机器：redis-cli --cluster add-node ip:port newIp:newPort

分配槽位：redis-cli --cluster reshard ip:port

![image-20200620141014908](https://gitee.com/lwj156/lwj-study/raw/master/image/redis/image-20200620141014908.png)

**客户端请求方式**：若我们通过客户端去访问数据，在redisA中获取redisB中的数据，那么请求会进行重定向到目标机器当中

**主从切换方式**：cluster模式中，每个mater节点也可配置slave节点作为主从。当主节点挂掉的时候，切换过程：

1. slave节点记录currentEpoch+1，广播failover消息给各个master节点
2. 各个master节点判断消息合法性，并发送ack消息给该slave节点
3. slave收到超过半数节点则成为master，广播通知其他集群节点

- cluster模式既有主从，又有类似哨兵的主从切换方式，因此不需要引入哨兵模式

**优势**：

1. 灵活性：可动态调整负责的slot范围
2. 可扩展性：可进行动态扩容和缩容
3. 高可用性：主从切换以及master集群

**缺点**：

1. 阻塞时间过长会被判断为failover
2. 主从数据无法保证强一致性



数据倾斜问题：集群部分节点数据量过大

1. 前期业务涉及避免高热度数据集中分布
2. 手动调整slot，将数据分摊到不同redis节点当中