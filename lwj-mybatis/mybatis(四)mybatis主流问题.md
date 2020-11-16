# mybatis

## mybatis主流问题

### #{}和${}的区别：

1. #{}是生成预编译sql（？替换占位符）：PreparedStatement的set来赋值

2. ${}就是替换变量的值，直接的静态string替换，可能88会有sql注入

### mybatis dao层如何找到xml文件中对应的sql：

约定里，接口的权限定名称以及方法名要对应xml中的namespace以及MappedStatement的id值。那么根据反射就能获取到接口名+方法名就能获取对应的mybatis解析xml文件的MappedStatement对象。里面包括了sql语句。方法调用是通过代理模式去做的

### mybatis如何分页：

分为物理分页和逻辑分页

1. 物理分页：即通过插件修改执行的sql语句，添加limit等。常用的pageHelper插件就是物理分页，原理是通过代理Executor去修改sql语句，分页参数存储在ThreadLocal当中
2. 逻辑分页：内存分页，查出数据后在进行分页处理。mybatis通过RowBounds对象进行逻辑分页

### mybatis插件原理：

mybatis可通过插件进行干预的4个对象有：ParameterHandler、ResultSetHandler、StatementHandler、Executor

实际执行的是mybatis的Interceptor接口，需要实现的方法有3个



### mybatis延迟加载原理：

mybatis关联查询有association和collection：区别在与一对一和一对多

延迟加载的原理：可对比hibernate的延迟加载

mybatis的延迟加载时通过代理模式。当获取对象的成员属性的时候，若为空，会去执行事先存储的sql



### mybatis的xml映射文件当中：xml文件的id是否能一致：

根据mybatis源码存储MapperStatement对象的map来看，key是nameSpace+id组成。因此相同id不影响数据存入



### mybatis批处理原理：

mybatis的3个处理器：simple、reuse（重复利用Statement）、batch，其中批处理采用batchExecutor，通过一个对象存储批处理sql

List<Statement> statementList



### mybatis可不可以映射成一些特殊类型，枚举、json等：

mybaits可以自定义typeHandler的，通过集成BaseTypeHandler可以实现