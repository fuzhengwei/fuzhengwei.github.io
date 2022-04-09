

# JUC锁核心类AQS解析

## 简介

AQS是一个用来构建锁和同步器的框架，使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的ReentrantLock，Semaphore等，它使用一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作，它是我们实现大部分同步需求的基础。

AQS的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态，AQS的涉及是基于模板方法模式的，它内部定义好了所有跟锁相关的底层操作细节与算法骨架，只保留了几个方法让子类重写，子类只需要重写几个特定的方法，就可以实现一个同步组件的功能。

通过使用AQS来构建我们自己的同步组件，可以帮助实现者简化同步组件的实现方式，将同步状态管理、线程的排队、等待与唤醒机制等底层操作封装在AQS里面，并通过模板方法定义好AQS的算法骨架，使得AQS基本上算的上是开箱即用

## 使用示例

AQS提供了五个方法让子类重写

![image-20220409141747298](http://img.jjjzzzqqq.top/image-20220409141747298.png )

```Java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。

```

如果我们想实现一个独占锁或者是实现一个共享锁，我们只需要重写其中两个方法即可。

而在我们重写这些方法的时候，我们可以用到AQS定义好的几个模板方法，来对同步状态进行操作。

![img](http://img.jjjzzzqqq.top/2019081909351520.png)

下面通过一个简单的自定义同步组件，来展示一下AQS的使用方法

```Java
/**
 * 通过AQS实现一个独占锁
 *
*/
public class Mutex implements Lock {

    /**
     * 静态代理模式
     * 代理类:Mutex
     * 被代理类:SYNC
    * */
    private final static Sync SYNC = new Sync();


    @Override
    public void lock() {
        SYNC.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        SYNC.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return SYNC.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return SYNC.tryAcquireNanos(1,unit.toNanos(time));
    }

    @Override
    public void unlock() {
        SYNC.release(1);
    }

    @Override
    public Condition newCondition() {
        return SYNC.newCondition();
    }

    private static class Sync extends AbstractQueuedSynchronizer {

        /**
         * 获取独占锁
         *
         */
        @Override
        protected boolean tryAcquire(int arg) {
            //CAS设置state从0到1
            if(compareAndSetState(0, 1)) {
                //设置独占线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        /**
         * 释放独占锁
         */
        @Override
        protected boolean tryRelease(int arg) {
            if(getState() == 0) {
                throw  new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        /**
         * 是否处于占用状态
         * */
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }
}
```



## 实现分析

### 原理概览

AQS的核心思想是，如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。而这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中。

AQS使用一个Volatile的int类型的成员变量来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，通过CAS完成对State值的修改。

### 同步队列

AQS中的队列是一个FIFO虚拟双向队列，AQS通过将每条请求共享资源的线程封装成一个节点来实现锁的分配。

![img](http://img.jjjzzzqqq.top/7132e4cef44c26f62835b197b239147b18062.png)



#### 同步队列节点

我们先看一下AQS这个内部类Node都包括哪些信息

![image-20220409145941781](http://img.jjjzzzqqq.top/image-20220409145941781.png)

| 方法和属性值  | 含义                                                         |
| :------------ | ------------------------------------------------------------ |
| waitStatus    | 当前节点在队列中的状态                                       |
| thread        | 表示处于该节点的线程                                         |
| prev          | 节点的前置指针                                               |
| next          | 节点的后置指针                                               |
| nextWaiter    | 指向下一个处于CONDITION状态的节点（与Condition队列实现有关） |
| predecessor() | 返回前驱节点()， 没有的话抛出NPE                             |

还有几个大小的变量，很明显是一些常量，int类型的常量是waitStatus状态的枚举值，Node类型的常量代表线程两种锁的模式。

![image-20220409150927400](http://img.jjjzzzqqq.top/image-20220409150927400.png)

线程两种锁的模式

| 模式      | 含义                                             |
| --------- | ------------------------------------------------ |
| SHARED    | 为了表明一个节点处于共享模式等待锁（大白话翻译） |
| EXCLUSIVE | 表示线程正在以独占的方式等待锁                   |

waitStatus的枚举值

| 枚举      | 含义                                           |
| --------- | ---------------------------------------------- |
| 0         | 当一个Node被初始化之后的默认值                 |
| CANCELLED | 为1，表示线程获取锁的请求已经取消了            |
| SIGNAL    | 为-1，表示线程已经准备好了，就等资源释放了     |
| CONDITION | 为-2，表示节点在等待队列中，节点               |
| PROPAGATE | 为-3，当前线程处在SHARED情况下，该字段才会使用 |

#### 入队

当一个线程获取锁失败后，会被构造成为一个节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：`compareAndSetTail(Node expect, Node update)`，它需要传递当前线程 “认为” 的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立连接。

#### 出队

队列的首节点是获取锁成功的节点，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点。

这个后继节点设置自己为首节点的过程中，原首节点就出队了。

由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法并不需要使用CAS来保证，它只需要将首节点设置成为原节点的后继节点并断开原首节点的next引用即可

### 同步状态State

了解了同步队列之后，我们来看看AQS关于同步状态State的操作

（注意不要搞混了，state是锁的状态，waitState是等待线程的状态）

![image-20220409151602819](http://img.jjjzzzqqq.top/image-20220409151602819.png)

AQS提供了三个方法来操作该同步状态

| **方法名**                                                   | **描述**             |
| ------------------------------------------------------------ | -------------------- |
| protected final int getState()                               | 获取State的值        |
| protected final void setState(int newState)                  | 设置State的值        |
| protected final boolean compareAndSetState(int expect, int update) | 使用CAS方式更新State |

我们可以通过修改State字段来实现多线程的独占模式与共享模式

实现思路

* 独占模式
    1. 获取锁时将state CAS(0,1)。CAS成功就代表获取锁成功，CAS失败就代表获取锁失败
    2. 释放锁时将state设置为0
* 共享模式
    1. 获取锁时判断state是否大于0
    2. 大于0就通过CAS自减获取锁成功，小于0就阻塞
    3. 释放锁时将state通过CAS自增





![img](http://img.jjjzzzqqq.top/27605d483e8935da683a93be015713f331378.png)



![img](http://img.jjjzzzqqq.top/3f1e1a44f5b7d77000ba4f9476189b2e32806.png)



### 源码分析

通过AQS源码来分析一下AQS对于锁的获取和释放流程。

#### 独占模式

##### 获取

通过调用AQS的acquire(int arg)方法可以获取同步状态，该方法对中断不敏感，也就是说后续对该等待线程进行中断操作时，线程不会从同步队列中移出。

```Java
public final void acquire(int arg) {
    //tryAcquire是我们自己重写的方法,返回的结果是获取锁成功还是失败
    if (!tryAcquire(arg) &&
        	//addWaiter先执行
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        	//获取锁失败且acquireQueued返回True就会中断该线程
            selfInterrupt();
    }
```

```Java
private Node addWaiter(Node mode) {
    	//创建Node
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            //将当前节点设置成尾节点
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
    	//CAS设置尾节点失败
        enq(node);
        return node;
    }
private Node enq(final Node node) {
    	//死循环自旋
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

此时该线程就加入了同步队列里面，我们在看一下之前的`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`方法

```Java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //自旋状态，当条件满足时退出
            for (;;) {
                final Node p = node.predecessor();
                //前置节点为head且尝试获取锁成功
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    //返回False,说明不需要中断
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                //取消获取资源
                cancelAcquire(node);
        }
}
// 当获取(资源)失败后，检查并且更新结点状态
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	//获取前驱结点的状态
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            //可以park
            return true;
        if (ws > 0) {
            //找到pred结点前面最近的一个状态不为CANCELLED的结点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            //赋值
            pred.next = node;
        } else {
            // 为PROPAGATE -3 或者是0 表示无状态,(为CONDITION -2时，表示此节点在condition queue中) 
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
}
// 进行park操作并且返回该线程是否被中断
private final boolean parkAndCheckInterrupt() {
    //park当前线程，并且设置了blocker
    LockSupport.park(this);
    return Thread.interrupted(); // 当前线程是否已被中断，并清除中断标记位
}
```

总结

![image](http://img.jjjzzzqqq.top/java-thread-x-juc-aqs-2.png)

1. acquire获取锁
2. 失败后加入同步队列
3. 执行acquireQueued方法
4. 自旋，退出循环的条件是前置节点为Head节点且获取锁成功
5. 否则会被LockSupport的park方法阻塞住等待唤醒，并且还会更新节点状态以及检查中断状态

##### 释放

释放锁逻辑在release方法中

```Java
public final boolean release(int arg) {
    //子类重写，返回的结果为释放成功还是失败
    if (tryRelease(arg)) { // 释放成功
        // 保存头节点
        Node h = head; 
        if (h != null && h.waitStatus != 0) // 头节点不为空并且头节点状态不为0
            unparkSuccessor(h); //唤醒头节点的后继结点
        return true;
    }
    return false;
}
```

```Java
// 释放后继结点
private void unparkSuccessor(Node node) {
    // 获取node结点的等待状态
    int ws = node.waitStatus;
    if (ws < 0) // 状态值小于0，为SIGNAL -1 或 CONDITION -2 或 PROPAGATE -3
        // 比较并且设置结点等待状态，设置为0
        compareAndSetWaitStatus(node, ws, 0);
    // 获取node节点的下一个结点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) { // 下一个结点为空或者下一个节点的等待状态大于0，即为CANCELLED
        // s赋值为空
        s = null; 
        // 从尾结点开始从后往前开始遍历
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) // 找到等待状态小于等于0的结点，找到最前的状态小于等于0的结点
                // 保存结点
                s = t;
    }
    if (s != null) // 该结点不为为空，释放许可
        LockSupport.unpark(s.thread);
}

```

后继节点此时就会被唤醒，继续在acquireQueued方法中执行自旋的逻辑，成功后则后续节点成功获取到了锁。

#### 共享模式

##### 获取

主要逻辑在acquireShared方法里

```Java
public final void acquireShared(int arg) {
    	//子类实现的方法，返回的值为int类型，> 0 就说明能够获取到同步状态
        if (tryAcquireShared(arg) < 0)
            //获取失败
            doAcquireShared(arg);
}
private void doAcquireShared(int arg) {
    	//添加到同步队列中
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            //自旋
            for (;;) {
                final Node p = node.predecessor();
                //前置节点为头节点
                if (p == head) {
                    //尝试获取锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //阻塞以及检查中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
}
```

1. 尝试获取锁，失败则添加到同步同列中
2. 自旋检查前置节点是否为头节点，是的话尝试获取锁
3. 获取锁失败之后会阻塞且检查中断
4. 等待unpark方法唤醒继续执行获取锁逻辑

##### 释放

```Java
 public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
    	//由于共享锁会有多个线程获得锁，所以释放锁的时候要自旋防止CAS失败
    	//独占锁由于只会有一个线程获得锁，所以释放锁的时候直接CAS state就行了
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

我们可以发现与共享锁的释放大体上一样，只不过需要自旋释放。

## 总结

AQS是一个功能很强大队列同步器类，基本上JUC下的多线程工具类都是由它实现，掌握了AQS的原理之后，大家就可以去研究一下ReentrantLock、Semaphore、CountDownLatch等是怎么实现的了。

![image-20220409191759888](http://img.jjjzzzqqq.top/image-20220409191759888.png)

