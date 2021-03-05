# JVM

> 文档地址：https://docs.oracle.com/javase/8/

<img src="JVM.assets/image-20210120184734267.png" alt="image-20210120184734267" style="zoom: 33%;" />

## 基本原理

### 编译过程

<img src="JVM.assets/image-20210120223112189.png" alt="image-20210120223112189" style="zoom: 50%;" />

#### 类加载过程

> 类装载模式：双亲委派模式
>
> 若自定义类加载自己写的java.lang的类，会抛出异常
>
> java.lang.securityException

<img src="JVM.assets/image-20210120193652853.png" alt="image-20210120193652853" style="zoom:33%;" />

<img src="JVM.assets/image-20210120195753517.png" alt="image-20210120195753517" style="zoom:50%;" />

**双亲委派如何破坏**：



### 运行时数据区

![image-20210121223727759](https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210121223727759.png)

#### 方法区（Method Area）

**对象头信息**

<img src="JVM.assets/image-20210121194859964.png" alt="image-20210121194859964" style="zoom:50%;" />

#### <img src="JVM.assets/image-20210121221413564.png" alt="image-20210121221413564" style="zoom:50%;" />