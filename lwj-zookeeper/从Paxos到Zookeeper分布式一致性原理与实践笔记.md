# zookeeper

## 一致性算法

常见的分布式一致性算法有：2pc、3pc、paxos算法（zab、raft）等

### 2pc

**角色**：协调者（Coordinator）、参与者（Participant）

**事务提交/回滚的过程**：

<img src="%E4%BB%8EPaxos%E5%88%B0Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E7%AC%94%E8%AE%B0.assets/image-20200630161308202.png" alt="image-20200630161308202" style="zoom:50%;" />

<img src="%E4%BB%8EPaxos%E5%88%B0Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E7%AC%94%E8%AE%B0.assets/image-20200630161327528.png" alt="image-20200630161327528" style="zoom:50%;" />

优点：原理简单、实现方便

缺点：

1. 同步阻塞：等待协调者指令过程中，参与者都是阻塞状态
2. 单点问题：协调者的可用性
3. 数据一致性：协调者放松commit请求过程中，部分协调者因网络等原因未收到，事务未提交
4. 参与者故障：参与者的可用性导致协调者未收到响应。通过超时中断事务



### 3pc

<img src="%E4%BB%8EPaxos%E5%88%B0Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E7%AC%94%E8%AE%B0.assets/image-20200630161805318.png" alt="image-20200630161805318" style="zoom:50%;" />

优点：减少了参与者阻塞的范围

缺点：参与者如果没收到doCommit请求，一定时间后还是会执行事务提交

将多个类用`Chain`链起来，暴露一个方法给Handler使用

### Paxos算法

![image-20200701194032549](%E4%BB%8EPaxos%E5%88%B0Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E7%AC%94%E8%AE%B0.assets/image-20200701194032549.png)

参考地址：https://baozh.github.io/2016-03/paxos-learning/

### zab协议

两种基本模式：

1. 崩溃恢复：当leader节点网络中断或者宕机，zab协议进入恢复模式。当leader选举出来并有过半的机器同步数据之后，退出恢复模式。当有新节点加入的时候也会进入恢复模式
2. 原子广播（简化版的2pc）
   1. leader收到消息，生成带有zxid的消息作为提案（proposal）发给follow节点的对列当中。follow写入磁盘后返回ack
   2. leder收到过半ack后会发送commit指令`区别于2pc的阻塞等待`（可能会有数据一致性问题。部分follow没回复），同时本地执行该消息。follow收到commit后提交该消息

## Leader选举



![image-20200701194309907](%E4%BB%8EPaxos%E5%88%B0Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E8%B7%B5%E7%AC%94%E8%AE%B0.assets/image-20200701194309907.png)

网图：仅参考用

**选举过程**：每个server发出投票，默认投自己，发送给其他server（）