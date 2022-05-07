# Redisson入门

## 概念

Redisson是一个在**Redis**的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的**分布式的Java常用对象**，还提供了许多**分布式服务**。其中包括(`BitSet`, `Set`, `Multimap`, `SortedSet`, `Map`, `List`, `Queue`, `BlockingQueue`, `Deque`, `BlockingDeque`, `Semaphore`, `Lock`, `AtomicLong`, `CountDownLatch`, `Publish / Subscribe`, `Bloom filter`, `Remote service`, `Spring cache`, `Executor service`, `Live Object service`, `Scheduler service`) Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

通过使用Redisson封装好的对象，程序员能像使用本地对象一样完成分布式的功能，可谓是十分便捷与强大。

## 使用

Redisson的使用很简单，主要有以下几个步骤

1. 导入Maven依赖
2. 配置Redisson Config
3. 使用Redisson的对象

详情参考官方Wiki，里面有十分详细的介绍

[Redisson]: https://github.com/redisson/redisson/wiki/1.-%E6%A6%82%E8%BF%B0

## 原理

分布式服务，本质就是多台机器操作同一个服务，从而达到分布式协调的效果，对比本地服务与分布式服务的区别，我们可以发现其实只不过本地服务针对是单机应用，而分布式服务针对的是多台分布式应用，那么我们就可以借助Redis这个媒介，借鉴本地服务的实现，实现我们的分布式服务，同时，在实现分布式服务时，一定要注意多线程并发操作下的线程安全问题，以及各种情况下的宕机问题。

### 可重入锁实现

我们先来理一理Java中的ReentrantLock有哪些实现要点

1. 锁资源：AQS中一个整型的state变量（通过对该变量的增减操作代表获取锁和释放锁）
2. 双向队列：一个保存了所有阻塞线程的队列，通过该队列可以达到阻塞和唤醒线程的操作

通过以上这两个主要部分，就可以实现ReentrantLock，那么我们现在换成基于Redis实现的分布式锁，就需要在Redis中寻找这么一种结构，通过操作Redis中的数据，来实现分布式锁。

Redisson如何实现的这两个功能

1. 锁资源：一个Redis中的HashKey，Key代表锁名称，Filed是`UUID:线程ID`来标识获取锁的线程，Value代表该线程获取锁的次数，用来实现可重入的功能
2. 双向队列：通过Redis的发布订阅来实现，如果获取锁失败，就会订阅Redis对应的频道并且根据返回的ttl将对应的线程阻塞对应的时间，当一个线程释放锁的时候，会通过对应的频道发布一条信息，然后之前获取锁失败的线程就会收到这一条信息，然后会再次尝试获取锁

### 看门狗实现

```Java
org.redisson.RedissonLock#tryAcquireAsync
```

<img src="http://img.jjjzzzqqq.top/17126dc70e9ca4e5~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp" alt="img" style="zoom:67%;" />

这里的 ttlRemaining 就是经过 lua 脚本后返回的值。经过前面我们知道了，**当加锁成功或者重入成功后会返回 null**。进入这个核心方法：

<img src="http://img.jjjzzzqqq.top/17126dd7c294f259~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp" alt="img" style="zoom:67%;" />

很明显，从上面标注的数字可以看出来：

①：这是一个任务。

②：这任务需要执行的核心代码。

③：该任务每 internalLockLeaseTime/3 ms 后执行一次。而  internalLockLeaseTime 默认为 30000。所以该任务每 10s 执行一次。

接着我们看一下 ② 里面执行的核心代码是什么：

![img](http://img.jjjzzzqqq.top/17126de38c46f2bc~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

这个 lua 脚本，先判断 UUID:threadId 是否存在，如果存在则把 key 的过期时间重新设置为 30s，这就是一次续命操作。

internalLockLeaseTime这个时间是可以修改的，比如我们想要修改为 60s，就这样：

![img](http://img.jjjzzzqqq.top/17126decdeaca24e~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

接下来，我们看看这个 task 任务是怎么实现的。

![img](http://img.jjjzzzqqq.top/17126df9f89a9ffd~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

可以看到，这个 Timeout 是 netty 包里面的类。

这个 task 任务是基于 **netty 的时间轮**做的。

面试官追问你：啥是时间轮？

你又不知道。那你接着往下看。

### 时间轮又是啥？

你听到了时间轮，你首先想到了啥？

听到这个词，就算你完全不知道时间轮，你也该想到，轮子嘛，不就是一个环嘛。

网上随便一搜，你就知道它确实长成了一个环状：

![img](http://img.jjjzzzqqq.top/17126e01aa6d348b~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

它的工作原理如下：

图片中的时间轮大小为 8 格，**每格又指向一个保存着待执行任务的链表**。

我们假设它每 1s 转一格，当前位于第 0 格，现在要添加一个 5s 后执行的任务，则0+5=5，在第5格的链表中添加一个任务节点即可，同时标识该节点round=0。

我们假设它每 1s 转一格，当前位于第 0 格，现在要添加一个 17s 后执行的任务，则（0+17）% 8 = 1，则在第 1 格添加一个节点指向任务，并标记round=2，时间轮每经过第 1 格后，对应的链表中的任务的 round 都会减 1 。则当时间轮第 3 次经过第 1 格时，会执行该任务。

需要注意的是时间轮每次只会执行round=0的任务。

知道了工作原理，我们再看看前面说的 Timeout 类，其实就是 HashedWheelTimer 里面 newTimeout 方法的返回：

![img](http://img.jjjzzzqqq.top/17126e0888549cef~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

前面我们分析了，在 Redssion 实现看门狗功能的时候，使用的是 newTimeout 方法。该方法三个入参：

1.task，任务，对于 Redssion 看门狗功能来说，这个 task 就是把对应的 key 的过期时间重置，默认是 30s。

2.delay，每隔多久执行一次，对于 Redssion 看门狗功能来说，这个 delay 就是 internalLockLeaseTime/3 算出来的值，默认是 10s。

3.unit，时间单位。

其实，你发现了吗，**这个时候我们已经脱离了 Redssion 进入 Netty 了。**

我们只需要告诉 newTimeout 方法，我们要每隔多少时间执行一次什么任务就行。

那我们为什么不自己写个更加简单的，易于理解的 Demo 来分析这个时间轮呢？

比如下面这样的：

![img](http://img.jjjzzzqqq.top/17126e1c2522c2f9~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

上面的 Demo 应该是很好理解了。

到这里，**我们知道了看门狗是基于定时任务实现的，而这个定时任务是基于 Netty 的时间轮实现的。**



参考文章

[Why技术——Redisson](https://juejin.cn/post/6844904106461495303)

