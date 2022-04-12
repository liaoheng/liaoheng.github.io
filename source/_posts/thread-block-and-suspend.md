---
title: 线程(进程)的阻塞与挂起
date: 2022-02-24 14:10:13
tags:
---

## 阻塞与挂起

### 挂起(suspend)
> 是指在操作系统进程管理将前台的进程暂停(suspend)并转入后台的动作。将进程挂起可以让用户在前台执行其他的进程。挂起的进程通常释放除CPU以外已经占有的系统资源，如内存等。

### 阻塞(block)
>正在执行的进程由于发生某时间（如I/O请求、申请缓冲区失败等）暂时无法继续执行。此时引起进程调度，OS把处理机分配给另一个就绪进程，而让受阻进程处于暂停状态，一般将这种状态称为阻塞状态。

#### 两者的区别
**挂起**是一种主动行为，因此恢复也应该要主动完成；**阻塞**则是一种被动行为，恢复交由系统完成。而且挂起队列在操作系统里可以看成一个，而阻塞队列则是不同的事件或资源（如信号量）就有自己的队列。
![bi5vSe.md.jpg](https://s4.ax1x.com/2022/02/24/bi5vSe.md.jpg)
如图，运行中**挂起**，进入`禁止就绪`状态，*释放资源*，自身对换到外存中，不释放锁，不释放CPU。，比如sleep()。而运行中**阻塞**，进入`活动阻塞`状态，*释放资源*，释放锁，释放CPU，比如wait()。

也就是说**挂起**在运行状态中要比**阻塞**靠前，**挂起**是一直在内存中定期运行自己，检查是否可以跳出当前状态；**阻塞**是自己不在运行，需要第三方检查自己是否可以跳出当前状态，在一定条件下会操作系统可能将其移出内存保存到外存中。

### 代码中的关系
Java 的 [Thread.State](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.State.html) 中定义了线程的 6 种状态，分别如下：

1.  NEW 未启动的，不会出现在 dump 文件中 (可以通过 jstack 命令查看线程堆栈)
2.  RUNNABLE 正在 JVM 中执行的
3.  BLOCKED 被阻塞，等待获取监视器锁进入 synchronized 代码块或者在调用 Object.wait 之后重新进入 synchronized 代码块
4.  WAITING 无限期等待另一个线程执行特定动作后唤醒它，也就是调用 Object.wait 后会等待拥有同一个监视器锁的线程调用 notify/notifyAll 来进行唤醒
5.  TIMED_WAITING 有时限的等待另一个线程执行特定动作
6.  TERMINATED 已经完成了执行

#### sleep
也就是说这是`挂起(suspend)`

#### wait/notify(synchronized)
也就是说这是`阻塞(block)`

#### wait/notify和sleep的区别
`wait/notify`的作用域在锁对象，释放CPU，`sleep`的作用域在线程，持有CPU。它们都可以实现对线程的控制。

## 锁

**锁**可以说是我们最经常使用到`阻塞与挂起`的地方了。

### synchronized关键字

**synchronized(Object)**中`Object`就是作用域，自动获取获取锁，自动释放锁。

### wait/notify

**Object.wait/notify**中`Object`就是作用域，手动获取获取锁，手动释放锁。

**wait/nofity** 是通过 jvm 里的 **park/unpark** 机制来实现的，在 linux 下这种机制又是通过 **pthread_cond_wait/pthread_cond_signal** 实现。因此当线程进入到 wait 状态的时候其实是会放弃 cpu 的，也就是说这类线程是不会占用 cpu 资源，也不会影响系统加载。


### 参考
[操作系统——CPU和内存、挂起和阻塞](https://blog.csdn.net/weixin_37641832/article/details/83217104)
[分析 Java Object 中的 wait/notify](https://generalthink.github.io/2019/10/10/analysis-java-object-wait-notify/)
[synchronized(this/.class/Object),synchronize方法区别](https://www.jianshu.com/p/4c1ed2048985)
[Life Cycle of a Thread in Java](https://www.baeldung.com/java-thread-lifecycle)