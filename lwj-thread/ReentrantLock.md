# ReentrantLock

> ReentrantLock(jdk1.5+)：表示重入锁，它是唯一一个实现了Lock接口的类。重入锁指的是线程在获得锁之后，再次获取该锁不需要阻塞，而是直接关联一次计数器增加重入次数



## 使用

```java
public static void main(String[] args) {
  Lock lock=new ReentrantLock();
  // 注意lock加锁要在try..catch结构体外面
  // 原因是如果lock加锁失败被异常捕获后，在执行unlock操作
  // 而当前线程并未获得锁，会抛出IllegalMonitorStateException
  lock.lock();
  try {
    // 业务逻辑
  } catch (Exception e) {
    e.printStackTrace();
  }finally {
    lock.unlock();
  }
}
```



## 核心源码解析

### 时序图

![image-20210304193135591](https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210304193135591.png)

### 类图

![image-20210304192345615](https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210304192345615-4925962.png)



## Reentrantlock#lock()

ReentrantLock是基于aqs双向队列来实现的

![image-20210305114346184](https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210305114346184-4915863.png)

> 入口方法有两个实现类，对应公平锁（FairSync）和非公平锁（NonfairSync）

![image-20210304193508611](https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210304193508611.png)

以非公平锁（NonfairSync）为例

### NonfairSync#lock()

```java
final void lock() {
  // 与公平锁的区别：不管是否有线程等待队列，先cas抢占
  if (compareAndSetState(0, 1))
    // 抢占锁成功，记录占有锁线程信息
    setExclusiveOwnerThread(Thread.currentThread());
  else
    // cas失败，走抢占锁逻辑
    acquire(1);
}
```

#### AQS#acquire()

> 该方法主要过程：
>
> 1. 尝试获得锁，成功则返回，失败则加入AQS队列尾部
> 2. 接点`自旋`尝试获得锁，过程中需要阻塞

```java
public final void acquire(int arg) {
  // tryAcquire()尝试获取独占锁
  // addWaiter()失败则加入aqs队列
  // acquireQueued()自旋尝试获得锁
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

`Node.EXCLUSIVE`代表该节点是独占模式下的等待节点，独占模式即只有一个线程可以获得锁

**node节点状态以及waitStatus值**：

```java
static final class Node {
        /**
         * 共享模式等待标记
         */
        static final Node SHARED = new Node();
        /**
         * 独占模式等待标记
         */
        static final Node EXCLUSIVE = null;
        /**
         * 代表队列超时或者中断
         */
        static final int CANCELLED =  1;
        /**
         * 如果该节点的下一个节点是阻塞状态，则该节点是SIGNAL状态
         */
        static final int SIGNAL    = -1;
        /**
         * 该状态节点会放在条件队列当中
         */
        static final int CONDITION = -2;
        /**
         * 处于共享模式中，该节点可以唤醒传播
         * 处于该状态的节点都可以获得锁
         */
        static final int PROPAGATE = -3;
  			//...
}
```

**节点阻塞进入队列会将前一个节点的waitStatus设置为`SIGNAL`，节点释放的时候，会将原head节点的waitStatus设置为0，然后唤醒的节点替换成头节点**

### Sync#nonfairTryAcquire()

> tryAcquire()方法的具体实现
>
> 该方法就是尝试去获得锁的逻辑

```java
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  // state为0代表无锁状态，线程可以获取锁
  if (c == 0) {
    // cas成功的线程获得锁
    if (compareAndSetState(0, acquires)) {
      // 记录当前线程，用于线程重入的时候判断
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  // 如果同一个线程获得锁，则增加冲入次数
  else if (current == getExclusiveOwnerThread()) {
    // 同一个线程无并发安全性问题，因此不需要加锁或者cas操作state值
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

### AQS#addWaiter()

> 尝试获得锁失败后，会调用该方法讲node节点加入AQS队列中

```java
private Node addWaiter(Node mode) {
  // 把当前线程封装成node节点
  // 这里node构造方法的nextWaiter变量是节点独占标记
  Node node = new Node(Thread.currentThread(), mode);
  // tail为AQS队列的队尾节点，默认为空
  Node pred = tail;
  // pred不为空代表队列存在节点
  if (pred != null) {
    // 把当前节点加入队列并设置为tail节点
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  // 如果tail为空代表队列还未初始化
  // 初始化队列并把node加入队列
  enq(node);
  return node;
}
```

#### AQS#enq()

```java
private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) {
      // 初始化同步队列
      // 再一次自旋把节点加入队列
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      // 追加到队列末端
      node.prev = t;
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

### AQS#acquireQueued()

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      // 获取当前节点的prev节点
      final Node p = node.predecessor();
      // 如果是head，代表node可以抢占锁
      // 调用tryAcquire尝试获取锁
      if (p == head && tryAcquire(arg)) {
        // 抢占锁成功，把node设置为新的head节点
        // 原heade节点移除
        // 该操作只有获取锁的线程执行，不需要cas操作
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      // prev非head节点或者上一个线程还未释放锁，该线程挂起
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        // park挂起过程是否中断过
        interrupted = true;
    }
  } finally {
    // 异常情况则取消获得锁
    if (failed)
      cancelAcquire(node);
  }
}
```



#### shouldParkAfterFailedAcquire()

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  // 如果前一个节点为signal，那当前节点直接阻塞
  if (ws == Node.SIGNAL)
    return true;
  // waitStatus>0为CANCELLED状态，则移除该节点
  if (ws > 0) {
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    // 否则把前一个节点cas为signal状态
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

#### parkAndCheckInterrupt()

```java
private final boolean parkAndCheckInterrupt() {
  // 阻塞挂起的地方
  LockSupport.park(this);
  // 如果是中断唤醒，在下一次自选会再次挂起，等到获得锁后在响应中断
  // 返回中断标识，在aqs.acquire()方法体里面的selfInterrupt调用响应中断
  return Thread.interrupted();
}
```

#### selfInterrupt()

```java
static void selfInterrupt() {
  // 重新中断，因为acquireQueued是不会响应中断的
  Thread.currentThread().interrupt();
}
```

## Reentrantlock#unlock

> 入口Reentrantlock.unlock()

### AQS#release()

```java
public final boolean release(int arg) {
  // 尝试释放锁，释放锁成功
  if (tryRelease(arg)) {
    Node h = head;
    // 如果队列存在，唤醒后续节点
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

### Reentrantlock#tryRelease()

```java
protected final boolean tryRelease(int releases) {
  // state：重入状态，同一个线程加锁+1，解锁-1
  // 当state为0的时候，释放锁
  int c = getState() - releases;
  // 非持有线程进行解锁，抛出异常
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  boolean free = false;
  // 设置持有线程为空
  if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);
  }
  setState(c);
  // 如果只是减少重入次数，则锁还未释放
  return free;
}
```



### AQS#unparkSuccessor()

```java
private void unparkSuccessor(Node node) {
  // 释放锁时，总会把头节点的waitStatus设置为0
  int ws = node.waitStatus;
  if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
  // s就是需要唤醒的节点
  Node s = node.next;
  // 如果s为null或者s为cancel节点
  // 从尾部开始扫描，找到最近的一个可以唤醒的节点
  // 因为enq方法的操作是先设置prev，如果这时候unparkSuccessor从头节点扫描
  // 会导致遍历到此时enq节点前就中断了
  if (s == null || s.waitStatus > 0) {
    s = null;
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  // 唤醒下一个节点
  if (s != null)
    LockSupport.unpark(s.thread);
}
```

> 唤醒原本挂起的线程之后，线程会继续执行`AQS.acquireQueued()`方法中的自选逻辑，再次尝试获得锁



## 公平锁和非公平锁的区别

> 区别在于非公平锁存在两次插队的机会

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210305112005769.png" alt="image-20210305112005769" style="zoom:50%;" />

<img src="https://gitee.com/lwj156/picture/raw/master/image/thread/image-20210305112035743.png" alt="image-20210305112035743" style="zoom:50%;" />

相对于公平锁来说，多了一个!hasQueuedPredecessors()方法，用于保证锁抢占的顺序性。

### AQS.hasQueuedPredecessors()

```java
public final boolean hasQueuedPredecessors() {
  Node t = tail;
  Node h = head;
  Node s;
  // 判断当前线程是否是头节点的下一个节点
  // 保证锁竞争遵循队列的顺序性
  return h != t &&
    ((s = h.next) == null || s.thread != Thread.currentThread());
}
```



## Lock对比Synchronized

1. Lock更为灵活：可以自定义实现锁，实现读写锁、公平锁、非公平锁等
2. Lock锁可以控制锁的释放
3. 两者都是`重入锁`
4. Lock可以自定义实现锁



## AQS基本实现过程

### lock()

1. 判断当前锁标记。若无锁，则抢占

2. 若有锁。加入AQS队列，该队列会先自旋初始化，在加入，因此会有一个空节点

3. 加入成功后，会在尝试抢占

4. - 判断前一个节点是不是初始化的head的空节点，若是则尝试，这边有顺序性控制，因为如果头节点不是head的话，说明是后边加入队列的，则不走抢占逻辑，直接阻塞

5. 阻塞操作：

6. - waistatus>0（取消状态） 移除取消状态的node

7. - 把前一个节点waistatus设为signal后自旋再次进入阻塞（这一次仍然会尝试抢占锁）

8. - 返回中断标志位：用于响应中断

### unlock()

1. 尝试释放锁，释放锁成功
2. 唤醒队列的后续节点

