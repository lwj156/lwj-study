# mybatis

## 涉及到的设计模式

1. 装饰器模式
   1. Cache->PerpetualCache->LruCache
   2. Executor->SimpleExecutor->CacheExecutor

![image-20200629174258186](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629174258186.png)

2. 工厂方法模式
   1. SqlSessionFactory
   2. MapperProxyFactory
   3. ObjectProxy

3. 单例模式：
   1. ErrorContext：

![image-20200629155450299](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629155450299.png)

4. 代理模式：MapperProxy、Plugin、SqlSessionInterceptor等
5. 模版方法模式：BaseExecutor和SimpleExecutor等
6. 建造者模式：
   1. XMLConfigBuilder、XMLMapperBuilder、XMLStatementBuilder

7. 适配器模式：Log4j

8. 责任链模式：InterceptorChain

## PageHelper插件原理

pageHelper插件是作用在Executor层的代理，在构建Executor的时候生成代理类，若有多个插件，则嵌套生成代理类（责任链模式）

![image-20200629170726627](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629170726627.png)

接着我们看下mybatis中Plugin类构建的代理类逻辑

![image-20200629171031036](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629171031036.png)

可以看到执行的是inteceptor的intercept方法，而inteceptor就是配置文件中声明的interceptor参数的实例化对象，在初始化xml解析的时候放入configuration当中InterceptorChain数组当中

![image-20200629171559690](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629171559690.png)

因此方法调用实际上是调用了PageInterceptor的intercept方法

该方法中持有传入的方法名、参数、sql等信息，就可以进行sql查询了，包括缓存处理也是在这里面，因为该方法持有存二级缓存的statement对象以及存一级缓存的Exetutor对象

**内部机制**：分页插件内部实际上执行了两个sql，一个count语句，一个获取实际数据的语句，在源码或者实操中都可以验证

![image-20200629173112848](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629173112848.png)

![image-20200629173025547](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629173025547.png)

> 那么分页参数是怎么来的呢，因为PageInterceptor物理分页，因此需要page等参数，执行sql前，若用分页插件，我们都会添加该语句PageHelper.startPage(1,2)

![image-20200629172546172](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629172546172.png)

可以看出分页参数是放在了ThreadLocal当中，那么有个问题，若ThreadLocal没有及时清除的话，会影响该线程后续sql查询。我们看下PageInterceptor类当中是否有释放逻辑

![image-20200629172706516](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629172706516.png)

![image-20200629172722422](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629172722422.png)

可以看出在afterAll方法里面执行了Threadlocal的remove操作，因此线程后续操作需要分页，需要重新执行PageHelper.startPage(1,2)

- 多个插件使用其实就是通过责任链模式生成了多层代理

## 如何实现自定义插件

> 看了PageHelper的原理分析，那么最快的了解插件原理的方式，便是查看PageHelper的写法，首先是继承interceptor接口

![image-20200629173315377](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629173315377.png)

然后是生命插件的作用域，因为插件可以作用在多个地方。PageHelper作用在Executor上面

![image-20200629173425010](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629173425010.png)

实现interceptor的三个接口方法(mybatis约定的接口实现类)

1. intercept：查询实际调用的方法
2. plugin：用于生成代理类的方法
3. setProperties：mybatis-config.xml声明插件可以指定properties参数，这边用于设值

![image-20200629173518189](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629173518189.png)

最后需要在mybatis-config.xml中注册插件



**应用场景**：

1. 日志输出
2. 自定义一个注解，指定分表逻辑
3. 数据加密：与mysql交互前后进行加密
4. 权限控制

## 整合spring

### SqlSessionFactoryBean源码分析

该类实现了FactoryBean、InitializingBean接口，在类初始化bean属性值设置完后会调用afterPropertiesSet方法

![image-20200629220105559](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629220105559.png)

![image-20200629220119018](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629220119018.png)

该方法就是原生mybatis中sqlsessionFactory的构建过程

接下来就是构建DefaultSqlSession，与原生mybatis不同的是，spring对其封装了

**常问的重点问题**：

> 为什么原生mybatis中DefaultSqlSession是非线程安全的？

主要问题还是在一级缓存当中

![image-20200629222221280](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629222221280.png)

如图，线程执行到到第一步的时候，localCache存入的是EXECUTION_PLACEHOLDER，之后执行到第二个红框内的时候才是list，那么我们在来看一下外层的判断

![image-20200629222354494](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629222354494.png)

可以看出，并发下，多个线程如果持有同一个Sqlsession，可能导致这里转换异常。还有就是若是同个Sqlsession下。一个线程插入未完成，另外个线程写入缓存，就会导致数据的不一致性。还有就是多个Sqlsession共享一个connection的线程安全性

![](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629230636204.png)

并发下，若每个请求创建一个Sqlsession是没有线程安全性问题，这样就导致Sqlsession无法作为单例，也没法复用的问题

spring做的处理就是持有一个单例sqlsessionTemplate，然后通过threadLocal维护线程的sqlsession

![image-20200629230757216](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629230757216.png)

mybatis原生不用SqlSessionManager的原因是：未集成spring的情况下管理动态代理效率不高，因此直接分开管理。每次请求都自己创建sqlsession。

SqlSessionTemplate更为高效，有做引用计数等复用

### SqlSessionTemplate源码分析

1. spring是通过构造注入SqlSessionTemplate的

```
<bean id="sqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
	<constructor-arg name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

![image-20200630094032135](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200630094032135.png)

2. 可以看到SqlSessionTemplate当中的select()都是通过sqlSessionProxy去操作的

**那么我们看sqlSessionProxy是如何生成的**：

![image-20200630094131714](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200630094131714.png)

3. 在SqlSessionTemplate的构造方法可以看出，实际操作是通过SqlSessionInterceptor这个代理类去处理的，SqlSessionInterceptor.invoke方法中通过getSqlSession方法获取SqlSession

![image-20200630094829485](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200630094829485.png)

4. TransactionSynchronizationManager.getResource里面调用的doGetResource方法。可以找到存储SqlSessionHolder的对象resources，通过ThreadLocal保证线程安全

![image-20200630095334296](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200630095334296.png)



## basedao的基本原理

basedao的两种实现思路

1. 若是自己定义basedao然后xml的save、insert、delete是自动生成的，底层可通过sqlsessionTemplate去获取操作的

> 那么我们要如何拿到sqlsessionTemplate这个对象呢：

mybatis提供了一个扩展类SqlSessionDaoSupport，里面持有sqlsessionTemplate对象

![image-20200630095918112](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200630095918112.png)

有了这个。我们就可以自定义basedao<T>通过反射去组装statementId，然后通过SqlSessionTemplate去查询

2. 若是基本操作的xml也不写的话，可结合@SelectProvider和@RegisterMapper去做

![image-20200629194959269](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629194959269.png)

![image-20200629195027369](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200629195027369.png)

原理上是自己组成sql语句，mybatis解析的话跟xml解析的时候一致的。这种方式的sql底层也是通过调用@SelectProvider中type对象中相应的方法生成的sql语句。



总结：

1. 插件的原理就是通过代理模式去作用在mybatis不同组件上
2. SqlSessionTemplate线程安全是通过线程池管理每个线程下的SqlSession
3. 原生mybatis的DefaultSqlSession线程不安全：因为若多个线程持有同一个SqlSession，就没法控制session.commit事务提交的时机。当然还有其他创建的线程安全性问题
4. basedao的原理：两种方式，通过持有SqlSessionTemplate去封装；或者通过@SelectProvider加上反射去实现sql语句的生成



---

穷尽所有可以提升自己的机会。---共勉