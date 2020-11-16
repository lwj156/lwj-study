# mybatis

> 源码分析大部分说明直接注释在代码当中

## mybatis的基本使用

![image-20200628210318171](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628210318171.png)

## SqlSessionFactory创建源码分析

**时序图**：

![image-20200628210431940](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628210431940.png)

首先几个关键注册点：

1. 存储sql语句和statementId的对象org.apache.ibatis.session.Configuration#addMappedStatement

   ![image-20200628211104779](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628211104779.png)

![image-20200628211214749](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628211214749.png)

可以看出key的结构以及xml文件中我们定义的sql语句存储在该对象当中。

2. 其他相关配置项信息也都存储在Configuration当中



构建完DefaultSqlSessionFactory后，需要构建SqlSession

## SqlSession构建源码分析

入口就是sqlSessionFactory.openSession()

主要功能是构建Executor，返回DefaultSqlSession

![image-20200628213108709](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628213108709.png)

**Executor**构建：

![image-20200628213310900](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628213310900.png)

**需要注意的是**：配置了二级缓存，那么生成的executor是CachingExecutor，需要区分的是，二级缓存是存储在MappedStatement当中，不要以为是存储在CachingExecutor当中了

## getMapper源码分析

入口还是在最上方的示例代码当中：sqlSession.getMapper(UserMapper.class)

![image-20200628214340244](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628214340244.png)



其实getmapper获取的对象并不是我们生命的mapper接口实例，而是一个代理类

## mapper执行方法过程源码分析

入口于示例代码当中userMapper.selectUser(1)

由于mapper对象是一个代理类，因此执行的是代理类的invock方法

![image-20200628215012263](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628215012263.png)

可以看出，若mapper接口中没有声明default方法的话，默认会执行到mapperMethod.execute方法当中，继续往下看

![image-20200628215156026](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628215156026.png)

根据不同的语句类型会走到不同的逻辑，其中select方法还是执行到了sqlSession的selectOne方法，command.getName其实就拿到了之前存储sql的map中MappedStatement对应的key（mapperInterface.getName() + "." + methodName组成），那么就可以拿到MappedStatement中对应的sql语句了，至于command中的name如何组成的，可参照下面截图，继续往下走

![image-20200628220146848](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628220146848.png)

![image-20200628215311214](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628215311214.png)

此刻到了executor的query方法，中间会有二级缓存以及一级缓存的判断逻辑，若两个逻辑都没有值，那么会走到mysql当中去查询

![image-20200628215452860](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628215452860.png)

到这其实就是熟悉的jdbc操作了，在handler.query中可以更清晰的看到

![image-20200628215603525](https://gitee.com/lwj156/picture/raw/master/image/mybatis/image-20200628215603525.png)

到这我们就可以看到，mybatis的执行原理

> 其实也就是对jdbc的封装
>
> mapper接口其实就是一层代理接口，执行的时候其实执行的是代理类中的逻辑。
>
> 这就解释为啥mapper是个接口，没有对应的实现却能执行的原因
>
> 最底层也是执行jdbc的操作



到这mybatis主要执行流程就是这样了，可自行通过debugger调试会清晰很多

由于注释都写在代码里面了，贴的代码多的话见谅

---



面对高度你需要展现的是饥饿，不是恐惧和吹嘘。---共勉



