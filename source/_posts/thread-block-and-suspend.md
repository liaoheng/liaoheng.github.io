---
title: 线程(进程)的阻塞与挂起
date: 2022-02-24 14:10:13
tags:
---

# 阻塞与挂起

## 挂起(suspend)

> 是指在操作系统进程管理将前台的进程暂停(suspend)并转入后台的动作。将进程挂起可以让用户在前台执行其他的进程。挂起的进程通常释放除CPU以外已经占有的系统资源，如内存等。
> 

## 阻塞(block)

> 正在执行的进程由于发生某时间（如I/O请求、申请缓冲区失败等）暂时无法继续执行。此时引起进程调度，OS把处理机分配给另一个就绪进程，而让受阻进程处于暂停状态，一般将这种状态称为阻塞状态。
> 

## 两者的区别

**挂起**是一种主动行为，因此恢复也应该要主动完成；

**阻塞**则是一种被动行为，恢复交由系统完成。而且挂起队列在操作系统里可以看成一个，而阻塞队列则是不同的事件或资源（如信号量）就有自己的队列。

![](https://pic.liaoheng.me/hc-pic/2024/02/3c148cfcf29b361b01d9002250f52873.jpg)

- 运行中**挂起**，进入`静止就绪`状态，*释放资源*，自身对换到外存中，不释放锁，也不释放CPU，比如sleep()；
- 而运行中**阻塞**，进入`活动阻塞`状态，*释放资源*，释放锁，释放CPU，比如wait()。

也就是说**挂起**在运行状态中要比**阻塞**靠前，**挂起**是一直在内存中定期运行自己，检查是否可以跳出当前状态；**阻塞**是自己不在运行，需要第三方检查自己是否可以跳出当前状态，在一定条件下也会由操作系统将其移出内存保存到外存中。

## Java中的线程与锁与其的关系

Java 的 [Thread.State](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.State.html) 中定义了线程的 6 种状态，分别如下：

1. NEW 未启动的，不会出现在 dump 文件中 (可以通过 jstack命令查看线程堆栈)
2. RUNNABLE 正在 JVM 中执行的
3. BLOCKED 被阻塞，等待获取监视器锁进入 synchronized 代码块或者在调用Object.wait 之后重新进入 synchronized 代码块
4. WAITING 无限期等待另一个线程执行特定动作后唤醒它，也就是调用Object.wait 后会等待拥有同一个监视器锁的线程调用 notify/notifyAll来进行唤醒
5. TIMED_WAITING 有时限的等待另一个线程执行特定动作
6. TERMINATED 已经完成了执行

### Thread.sleep

也就是说这是`挂起(suspend)`，作用域在线程，持有CPU。

### Object.wait/synchronized

也就是说这是`阻塞(block)`，作用域在锁对象，释放CPU。

> `wait/notify`是通过 JVM里的**park/unpark** 机制来实现的，在Linux下这种机制又是通过**pthread_cond_wait/pthread_cond_signal**实现。
> 

### Lock

也就是说这是`挂起(suspend)`，作用域在锁对象，持有CPU。

> Lock 用的是乐观锁机制。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。
> 

# 参考

- [操作系统——CPU和内存、挂起和阻塞](https://blog.csdn.net/weixin_37641832/article/details/83217104)
- [分析Java Object 中的 wait/notify](https://generalthink.github.io/2019/10/10/analysis-java-object-wait-notify/)
- [synchronized(this/.class/Object),synchronize方法区别](https://www.jianshu.com/p/4c1ed2048985)
- [Life Cycle of a Thread in Java](https://www.baeldung.com/java-thread-lifecycle)
- [Synchronized 和 lock 区别](https://xie.infoq.cn/article/4e370ded27e4419d2a94a44b3)