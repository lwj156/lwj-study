# mybatis

## 原生mybatis

先简单介绍下原生jdbc执行逻辑：

1. 注册驱动，获取connection连接
2. 创建statement（用于操作数据库）
3. statement.execute执行sql
4. 转换结果集（resultSet），关闭资源

**缺点**：

1. 只能顺序传参
2. 缺少统一查询的封装（平时用的basedao）
3. 缓存
4. sql硬编码

ORM框架对比：

**hibernate**：

1. 自定义sql灵活性低
2. 无法支持动态sql
3. 适合单表操作
4. sql优化比较难，基于实体映射的，难优化sql走索引等

**mybatis优势**：

1. sql和代码分离，看起来会清爽些，不过封装程度没有hibernate高
2. 灵活的动态sql和数据库连接池
3. 多级缓存
4. 重复sql的提取和自定义插件的使用

**核心对象**：

1. SqlSessionFactoryBuilder：存在于方法的局部，创建SqlSessionFactory后就没存在的意义
2. SqlSessionFactory：用于创建会话，作用域是应用作用域，即单例
3. SqlSession：会话，非线程安全，在一次请求或操作结束需及时关闭
4. Mapper：代理对象，用于发送sql，作用于方法内

![image-20200624200516537](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200624200516537.png)

**应用**：

1. typeHandlers：用于java类型到jdbc类型的转换

   > mybatis中自带的转换：如果需要自定义转换类型，可自己实现`BaseTypeHandler<T>`
   > 必须实现的4个抽象方法：set方法从Java类型转换成JDBC类型的，get方法是从JDBC类型转换成Java类

   ![image-20200627155701499](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200627155701499.png)

2. mybatis批量处理数据限制：max_allowed_packet 4M设置批量数据大小

3. mybatis设置批量执行器：

   `<setting name="defaultExecutorType" value="BATCH" />`或在创建的时候指定执行器类型，BatchExecutor底层也是对jdbc的封装，积攒一定的sql后再发送

4. collection和association的区别：
   collection用于一对多关系
   association用于一对一或者多对一关系

5. 逻辑翻页和物理翻页的区别：

   > 逻辑翻页：对象RowBounds，实际上是对ResultSet处理，即查询所有结果后分页

   > 物理翻页：通过sql语句(limit)进行翻页。可通过PageHelper插件实现，该插件原理就是在语句前面通过threadLocal设置分页信息，底层组装limit语句

6. **mybatis-config.xml文件的配置标签顺序按照官网，修改顺序可能导致解析失败**

   

### mybatis体系结构

![image-20200627160648045](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200627160648045.png)

1. 接口层：SqlSession接收到请求后回调用核心层对应模块处理

2. 核心层：

3. 1. 入参转成jdbc类型
   2. 解析sql语句，动态sql语句生成
   3. 执行sql语句
   4. 处理结果集，映射成java对象

4. 基础层：如数据源、缓存、日志、xml解析、反射、IO、事务等等

### mybatis缓存

1. 一级缓存：SqlSession层面的缓存，默认开启，不需要配置。

   存在于org.apache.ibatis.executor.BaseExecutor.PerpetualCache当中，底层也是一个hashMap对象存储：缓存同一个会话中一摸一样的SQL结果


![image-20200627204735099](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200627204735099.png)

其中缓存键：cacheKey的内部结构如下：

![image-20200627205524341](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200627205524341.png)

> 不能跨会话：会导致读取到脏数据（另外一个会话更新数据）

二级缓存：解决一级缓存的问题，namespace级别（Mapper级别）可以多个SqlSession共享，简单来说就是跨会话的一级缓存。

**session需要事务commit后才生效**

> 配置方式
>
> `<setting name="cacheEnabled" valie="true">`

![image-20200627202200117](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200627202200117.png)

1. 作用于一级缓存之前  默认开启  可手动关闭
2. 事务不提交，二级缓存不生效（源码中只有DefaultSqlSession调用commit之后才会调用 flushPendingEntries()刷新缓存）

3. 增删改操作会清空缓存：CachingExecutor.update()调用 flushCacheIfRequired(ms)，isFlushCacheRequired 就是标签的 flushCache 的值。而增删改操作的flushCache属性默认为true
4. 二级缓存场景：查询为主的应用中，比如历史数据的查询
5. 二级缓存脏数据：一个namepace对多表操作，会导致另外一个单表操作的namespace读取到脏数据，二级缓存是mapper级别的
6. 解决通过`<cache-ref>`标签指定缓存共享的namespace：导致多个表操作都更新缓存，缓存的意义不大
7. 自定义二级缓存：通过实现Cache定义二级缓存，如redis和ehcache。配置在cache的type属性中，同时也可以弃用危机缓存，用自定义缓存策略



## 参考地址：

1. 使用说明参考：[mybatis官网][https://mybatis.org/mybatis-3/zh/index.html] 