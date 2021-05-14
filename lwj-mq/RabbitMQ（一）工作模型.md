# RabbitMQ

**Erlang语言编写的mq**

对比java里面队列的实现：

> 1. 不能在分布式环境下使用
> 2. 没有持久化机制

**mq作用**

1. 异步
2. 解耦：业务不受下游服务影响
3. 削峰

**带来的问题：**

1. 可用性降低：增加一个运行服务，调用mq有失败的风险
2. 复杂度变高：代码复杂度，以及需要考虑消息丢失、重复等`数据一致性`问题

## 基本特性

1. 可靠性：持久化、应答、发布确认、高可用
2. 灵活的路由：通过交换机实现路由
3. 支持多客户端：多语言支持
4. 集群和扩展
5. 高可用队列：镜像队列实现复制
6. 权限管理：用户管理页面权限控制
7. 插件系统：插件及自定义插件支持
8. spring集成：spring对amqp进行了封装

> amqp是应用层协议，可对比http

### 工作模型

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210310192243553.png" alt="image-20210310192243553" style="zoom:50%;" />

1. Broker：mq服务器
2. Vhost（虚拟主机），实现资源的`隔离`和`权限`的控制，一个mq可以起多个Vhost。可以给不同的用户分配权限。实现业务系统之间的隔离。

> 不同的Vhost可以有同名的Exchange和Queue

3. Exchange（交换机）：实现消息的灵活`路由`，队列使用绑定键（Binding Key）和交换机创建绑定关系

> 交换机和队列，队列和消费者都是多对多

4. Queue（队列）：真正用来存储消息的，是独立运行的进程，通过Mnesia数据库存储

> 消息获取有两种模式：push模式（推送）和pull模式（拉取）

5. Channel（消息通道）：保持TCP长连接的创建和释放，减少资源消耗



### 路由方式

> 生产者发送消息会携带路由键

1. Direct（直连）：队列直连交换机，指定一个精确的绑定键，完全匹配路由键则发送到指定队列上
2. Fanout（广播）：绑定交换机的队列都会收到消息，队列不需要`绑定键`，消息也不需要路由键
3. Topic（主题）：绑定键可以使用通配符

> #：0个或者多个单词   *：不多不少一个单词



### 死信队列

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210311142838802.png" alt="image-20210311142838802" style="zoom:50%;" />

> 消息在一些情况下会变成死信（Dead Letter）

1. 被消费者拒绝且未设置重回队列
2. 消息过期
3. 队列达到最大消息数

**创建死信队列**：

死信交换机DLX(Dead Letter Exchange)和死信队列 DLQ(Dead Letter Queue)和正常的是一样的。

正常的队列在`创建`的时候可以指定一个死信交换机

## 基本使用（spring）

1. 声明交换机

```java
public DirectExchange(String name, boolean durable, boolean autoDelete, Map<String, Object> arguments) {
  super(name, durable, autoDelete, arguments);
}
```

- durable：是否持久化，Rabbitmq关闭后，`未持久化`的将被删除
- autoDelete：如果`没有绑定Queue`，将自动删除
- arguments：结构化参数

2. 声明队列

```java
public Queue(String name, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments) {
  Assert.notNull(name, "'name' cannot be null");
  this.name = name;
  this.durable = durable;
  this.exclusive = exclusive;
  this.autoDelete = autoDelete;
  this.arguments = arguments;
}
```

- exclusive：是否排他性队列，只能在声明它的connection当中使用
- autoDelete：是否自动删除，与队列消费者都断开则删除
- durable：代表队列在服务器重启后是否还存在
- arguments：队列的其他属性，详细见下图

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210311110637466.png" alt="image-20210311110637466" style="zoom:50%;" />

3. 建立绑定关系

```java
@Bean
public Binding testBinding() {
  return BindingBuilder.bind(testQueue()).to(testExchange()).with(TEST_ROUTY_KEY);
}
```

4. 发送消息

```java
public void produce(String exchange, String routingKey, Object payload) {
  try {
    MessageConverter messageConverter = rabbitTemplate.getMessageConverter();
    Message message = messageConverter.toMessage(payload, MessagePropertiesBuilder
                                                 .fromProperties(new MessageProperties())
                                                 .setContentType(MessageProperties.CONTENT_TYPE_SERIALIZED_OBJECT)
                                                 .build());
    rabbitTemplate.send(exchange, routingKey, message);
  } catch (Exception e) {
    LogUtil.error(log, LogTag.Error.ERROR_RABBIT_PRODUCER,
                  "exchange", exchange,
                  "routingKey", routingKey,
                  "payload", JSONUtil.toJSON(payload),
                  "error", LogUtil.exception2String(e));
  }
}
```

MessageProperties参数详情

```java
content_type ： 消息内容的类型
content_encoding： 消息内容的编码格式
priority： 消息的优先级
correlation_id：关联id
reply_to: 用于指定回复的队列的名称
expiration： 消息的失效时间
message_id： 消息id
timestamp：消息的时间戳
type： 类型
user_id: 用户id
app_id： 应用程序id
cluster_id: 集群id
Payload: 消息内容
```

5. 消费端

```java
@RabbitListener(queues = MQTestConfig.TEST_QUEUE)
public void testConsumer(@Payload String testMQRequest, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
  // 业务逻辑
}
```

- 注：MessageProperties和Queue都能指定消息的TTL，那么时间小的那个优先级高

6. 声明死信队列

```java
public Queue testDelayTaskQueue() {
  Map<String, Object> arguments = Maps.newHashMap();
  arguments.put("x-overflow", REJECT_PUBLISH);
  //指定死信交换器
  arguments.put("x-dead-letter-exchange", TEST_DEAD_EXCHANGE);
  //指定死信的路由键
  arguments.put("x-dead-letter-routing-key", TEST_DEAD_ROUTE_KEY);
  return new Queue(TEST_DELAY_QUEUE, true, false, false, arguments);
}
```

### 延迟队列的实现方式

1. Rabbitmq死信队列

> 缺点：时间梯度多（1分钟、2分钟等）需要建很多死信交换机和队列。可能存在一定的时间误差。
>
> 可能导致消息阻塞：前一条消息的`TTL`时间过长，导致后过期的消息无法投递

2. 先存储到数据库，在通过定时任务扫描

3. rabbitmq-delayed-message-exchange插件

## 常见问题

### 消息堆积

> 问题：当生产消息的速度远大于消费者消费的速度，会产生消息堆积

***队列控制长度属性：***

1. x-max-length：最大消息数，超过丢弃先入队的消息（队头的消息）
2. x-max-length-bytes：最大消息容量，超过丢弃队头消息

**内存控制：**rabbitmq.config配置内存阈值，默认40%

> [{rabbit, [{vm_memory_high_watermark, 0.4}]}].

超过则阻塞所有连接。



**磁盘控制：**类似内存控制，也是通过判断磁盘空间大小，当磁盘空间低于指定的值时(默认 50MB)，触发流控措施。

disk_free_limit.relative = 3.0 //比例

disk_free_limit.absolute = 2GB //大小



**业务处理：**

1. 增加消费者
2. 新建临时业务处理的消费者消费队列，不做业务处理，写入新的`临时队列`当中。等消费完积压数据后在慢慢处理临时队列的数据
3. 根据业务场景判断是否能在控制台`清理`积压消息



### 消费端OOM

> 队列会尽可能推送消息给消费者，消费者在本地缓存消息，可能导致OOM

配置超过一定数量消息未确认，则停止投递消息给消费者

spring.rabbitmq.listener.simple.prefetch=2



