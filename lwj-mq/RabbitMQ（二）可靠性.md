# RabbitMQ

## RabbitMQ集群

> RabbitMQ有两种集群模式：普通集群模式和镜像队列模式

1. 普通集群模式

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210313150559738.png" alt="image-20210313150559738" style="zoom:50%;" />

> 不同节点之间只同步`元数据`，队列数据不同步

元数据(包括队列名字属性、交换机的类型名字属性、绑定、vhost)

节点之间通过`转发`获取数据，无数据的节点起到路由作用

**缺点**：队列内的内容不复制，节点失效将导致队列不可用

**节点类型**：分为`内存`节点（用于读写）和`磁盘`节点（用于备份）

2. 镜像集群

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210313153433918.png" alt="image-20210313153433918" style="zoom:50%;" />

> 该模式小，消息内容会在镜像之间同步，系统性能下降，同步代价比较高

镜像同步后，每个节点数据一致，那么就需要指定`负载策略`来选择使用的节点

可用的负载均衡组件：HAProxy、Nginx

> 那么也同时需要考虑负载均衡的可用性，于是引入了Keepalived
>
> Keepalived之间使用VRRP`虚拟路由冗余协议`

1. HAProxy可以监控RabbitMQ节点，发生故障的剔除
2. Keepalived会自动选举一个mater节点（主路由器）：通过`广播心跳`实现。若一个节点挂了，则Backup节点成为Master
3. Keepalived-Master节点对外提供`虚拟`ip，供应用端使用



## RabbitMQ高可用

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210310192243553.png" alt="image-20210310192243553" style="zoom:50%;" />

各个环节的可用性分析

### 生产者发送消息到Broker

> 因为`网络`等原因导致消息发送失败或者生产者无法确认消息是否发送成功

发送端有两种模式可以使用

1. 事务（Transaction）模式：rabbitTemplate.setChannelTransacted(true)

只有收到服务端的Commit-OK指令才算提交成功，否则可以回滚。

**缺点**：只能串形发送，一条消息没发送完，无法发送下一条消息，影响性能

2. 确认（Confirm）模式：消息投递成功后，RabbitMQ会发送`Basic.Ack`给生产者

> - 通过调用channel.waitForConfirms()返回true，代表消息投递成功
> - 批量模式：channel.waitForConfirmsOrDie()没有异常，代表批量投递成功，缺点在于**一条失败则批量的消息都得重发**
> - 异步确认模式：通过回调方式确认（rabbitTemplate.setConfirmCallback）

### 交换机到队列之间的传输

> 由于队列不存在或者路由键错误

1. 通知生产者：rabbitTemplate.setReturnCallback
2. 消息路由到备份交换机：

```java
arguments.put("alternate-exchange","ALTERNATE_EXCHANGE");
channel.exchangeDeclare("TEST_EXCHANGE","topic", false, false, false, arguments);
```

### 队列持久化

> RabbitMQ由于宕机、故障等原因导致数据丢失

配置持久化到磁盘（队列、交换机、消息都可配置）

### 消息投递到消费者

> 传输过程或者消费者处理过程产生异常

RabbitMQ提供消费者的消息确认机制：手动或者自动发送ACK给服务端

配置方式：spring.rabbitmq.listener.simple.acknowledge-mode=manual

**消费端调用ACK手动确认的方式**：

```java
channel.basicAck(deliveryTag, false);
```

消息无法处理可以通过Basic.Reject拒绝单条，Basic.Nack批量拒绝

如果requeue为true则重新投递，若只有一个消费者，可能导致`无限循环`

### 消息的顺序性

> 如何保证多条消息的消费顺序跟投递顺序一致

当**一个队列只有一个消费者**的时候，需要保证顺序的消息发送到同一个队列上

### 消费者宕机

多次重发消息也没得到消费，可通过最终对账保持最终一致性

### 消息幂等性

> 1. 生产者Confirm模式未收到确认，发送重复消息
>
> 2. 消费者应答ack由于`网络原因`中断或者网络延迟，这条消息会重新发送给其他消费者消费，导致重复消费

**处理方式**：

1. 全局MessageId：每次消费前判断该id是否处理过，处理过的消息可存储在redis或者mysql当中
2. setNx：处理过的消息通过setNx存储，若失败则说明已处理过了

### 生产者如何感知消费者消费消息

生产者提供回调api接口

> 若长时间没响应生产者。可以自定义重发机制

## 消息堆积问题

> 生产者速度远大于消费者消费速度导致堆积

1. 临时多创建消费者消费
2. 根据业务场景判断是否可清空：RabbitMQ管理页面可清理
3. 新建一个消费服务，将堆积消息写入新的队列，等不再堆积后，在去消费