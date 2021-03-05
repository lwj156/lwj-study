# 并发编程（二）Synchronized锁



## 基本使用

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201119190928500.png" alt="image-20201119190928500" style="zoom: 33%;" />

1. 修饰实例方法：锁的对象为当前实例 syn void
2. 修饰静态方法：锁的对象为这个类 syn static void
3. 修饰代码块：看锁对象还是实例 syn(a.class)、syn(this)

**注**：不能用String常量、Integer、Long类型：因为Java自动封箱和解箱操作的原因



## 存储方式

**对象头加锁前后变化**：

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201203192833465.png" alt="image-20201203192833465" style="zoom: 25%;" />

**参考工具**：Java Object Layout

![img](https://gitee.com/lwj156/picture/raw/master/image/thread/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NjgyNjY1,size_16,color_FFFFFF,t_70-20210227170313180.png)

可以看出锁信息存储在对象的`对象头`当中：对应源码（markOop.hpp当中）

![img](https://gitee.com/lwj156/picture/raw/master/image/thread/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NjgyNjY1,size_16,color_FFFFFF,t_70-20210227170443541.png)

**对象头（marword）**：8字节对象头+4字节的指向指针（属于哪个类）

对象填充：`8`的倍数，缺少则填充

**不同类型的锁在对象头中存储的结构如下：**

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201203193114155.png" alt="image-20201203193114155" style="zoom:50%;" />



## 锁升级的过程

> Jdk1.6之前，基于重量级锁来实现

**无锁->偏向锁->轻量级锁->重量级锁**



### 偏向锁

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201119232307183.png" alt="image-20201119232307183" style="zoom: 33%;" />

> 存在的意义：大多数情况下不存在锁竞争，都是由同一个线程多次获取

通过cas实现原子性 

**关闭方式**：-usebiasedLock

**注**：线程1执行完毕后,不会主动去释放偏向锁。线程2竞争时，先判断偏向锁标记为1，然后在判断持有的线程1是否还存活，不存活，cas替换，存活则暂停升级。

**`偏向锁延迟`打开的作用**：JVM启动过程中会去对对象处理，这个时候会存在锁竞争问题，因此让偏向锁延迟打开。-XX:BiasedLockingStartupDelay默认4秒

偏向锁若`启用`，则创建的对象为匿名偏向锁对象1|01，此时记录的线程id为空



### 轻量级锁

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201119232325852.png" alt="image-20201119232325852" style="zoom: 33%;" />

偏向锁撤销后升级为轻量级锁，通过cas自旋获取锁

> 自旋的意义：绝大部分线程在获取线程后，在非常短的时间内会释放

**设置自旋次数**：preBlockSpin，默认10次

jdk1.6后引入`自适应自旋`：由JVM来控制（根据最近成功率调整自旋次数）

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201119232657942.png" alt="image-20201119232657942" style="zoom:33%;" />

> **轻量级锁加锁过程**：
>
> 1. 线程栈桢创建`LockRecord`并复制MarkWord中的锁记录
> 2. 将LockRecord中的Owner指针指向锁对象
> 3. 将锁对象的MarkWord替换为指向LockRecord的指针



### 重量级锁（Mutex）

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201122174651665.png" alt="image-20201122174651665" style="zoom: 33%;" />

基于对象监视器实现。

同步队列：objectMonotor.hpp中的`waitSet`队列

**偏向锁直接升级成重量级锁**：当获取偏向锁的对象调用了wait方法的时候会直接升级为重量级锁

**加锁过程**：

1. monitor依赖操作系统的`MutexLock(互斥锁)`来实现的，线程被阻塞后便进入内核（Linux）调度状态，这个会导致系统在`用户态与内核态`之间来回切换，严重影响锁的性能
2. 任意线程对Object的访问，首先要获取对象的monitor，获取失败加入同步队列，线程状态变为BLOCK，访问Object的前驱释放锁，唤醒队列中的线程重新尝试获得锁



### 锁重入

偏向锁和轻量级锁->线程栈中lockRecord+1

重量级锁->ObjectMonitor当中

### 锁消除

方法内部的锁只会被自己线程所用（如StringBuffer类的append操作），不可能被其他线程引用，则自动消除对象内部锁。

### 锁粗化

JVM检测多次操作对同一个对象加锁（如while操作），则会将锁范围加粗到方法体外，是这一串操作只进行一次加锁。



### wait、notify、notifyall

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20201122175826293.png" alt="image-20201122175826293" style="zoom:50%;" />

> 线程通信机制

wait：实现线程阻塞，释放锁当前的syn锁

notify/notifyall：唤醒被阻塞的线程

**notify唤醒线程后需要重新加入竞争锁队列，不会直接`获得`锁**

> wait和sleep的区别：
>
> wait`释放`锁和cpu资源，sleep不会释放锁资源



## cas机制

源码文件：unsafe.cpp

底层是lock前缀指令：lock cmpxchg指令

`lock`保证cmpxchg的原子性，单cpu不用加lock



### ABA问题

一个值从A->B->A，当你执行cas的时候，没有发现到变化

**解决方式**：通过版本号控制，1A->2B->3A这样，AtomicStampedReference实现类就是这个机制



## 用户态和内核态

分级ring0-ring3：ring0权限最高

sync重量级锁：需要从用户态到内核态的请求

> 偏向锁和轻量级锁属于用户态锁，不需要和操作系统打交道



## java内存模型（JMM）

<img src="Synchronized%E9%94%81.assets/image-20201120111519857.png" alt="image-20201120111519857" style="zoom:50%;" />

