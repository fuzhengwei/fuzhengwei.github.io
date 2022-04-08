# Java多线程编程技术实践

最近笔者面试的时候被面试官问到了有关Java线程池源码方面的问题，没有回答上来，所以准备好好研究一下Java线程池底层源码。当我打开JDK源码的时候，我整个人就愣住了，像极了刘姥姥进大观园，Java线程池源码中的一些涉及到多线程相关的API，我竟然已经完全忘记了这些方法的作用，于是便准备总结一篇Java多线程编程技术相关的内容，用以对自己所学知识的总结。

本文会总结常见的Java多线程基础练习和面试题，包括《Java多线程编程核心技术》、《Java并发编程的艺术》上的Code，以及常见的多线程笔试面试题。

## 基础

### Java线程状态变迁

这里笔者给出Java线程状态变迁一览图，之后提到的线程状态可在此查阅

<img src="http://img.jjjzzzqqq.top/image-20220329203558510.png" alt="image-20220329203558510" style="zoom:67%;" />

* NEW
* RUNNABLE
* WAITING
* TIMED_WATING
* BLOCKED
* TERMINATED

### Object类方法

#### wait()

调用该方法之后，当前线程会释放掉锁并且进入waiting状态。

#### notify()

该方法用来唤醒处于waiting状态的线程，调用了该方法之后，不会直接释放锁并唤醒线程，而是会等该线程将同步代码块执行完之后，也就是退出了syanchronized同步区域后，才会释放锁。

#### 总结

这两个方法是多线程编程技术中最基础的两个方法，使用这两个方法使需要注意，当调用某个对象的wait或者是notify方法前，该线程需要获得该线程的Synchronized锁，否则会出现IllegalMonitorStateException。



### Thread类方法

#### sleep(Time)

调用该方法之后，该线程会进入TIMED_WAITING状态，经过固定的时间后继续运行。

#### interrupt()

通过调用一个线程的 interrupt() 方法来中断该线程，但是中断不一定会生效，仅仅只是将该线程标记为中断状态，可以通过**interrupted()**方法和 **isInterrupted()** 判断该线程是否处于中断状态。那么中断什么时候会立即生效呢，如果该线程处于Blocked、TIMED_WATING、WATING状态，那么就会抛出 **InterruptedException**，从而提前结束该线程。但是不能中断 I/O 阻塞和 synchronized 锁阻塞。

那如果是不处于这些阻塞等待状态的线程我们如何使它结束呢？那么就要使用以下两个方法了。

#### interrupted()

通过interrupted方法，可以判断当前线程是否处于中断状态，处于的话，返回True，否则返回False，通过该方法，我们可以在一个循环执行的线程中判断线程是否中断了，如果是，就可以通过提前Return或者 Throw InterruptedException来结束该线程

#### isInterrupted()

该方法和上述方法类似，两个方法的区别如下：

* interrupted()方法：执行后会清除掉状态标志值的功能，会重新设置为false
* isInterrupted()方法：执行完不会清除状态标志



Code Demo

```Java
class MyThread implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    System.out.println("线程一直在执行");
                    if (Thread.interrupted()) {
                        throw new InterruptedException();
                    }
                }
            }catch (InterruptedException e) {
                Logger.getLogger(MyThread.class.getName()).info(Thread.currentThread().getName() + "线程结束了");
            }
        }
    }
```

通过这种方式，线程就能被interrupt()方法中断

我们再看看线程池中关于这两个方法的应用，线程池在这方面的应用比较复杂，笔者目前也不是太清楚，之后会详细的阅读线程池源码，总结这一部分的知识。

<img src="http://img.jjjzzzqqq.top/image-20220329214321642.png" alt="image-20220329214321642" style="zoom:67%;" />

```Java
//ThreadPoolExecutor.class
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```



#### stop()方法

该方法可以直接暴力使线程停止，该方法会强行终止线程，所以如果使用该方法，可能会导致业务数据处理不一致的情况，所以该方法目前已经废弃了。而它的实现原理就是在调用该方法时会抛出ThreadDeath Error，从而使线程直接退出。

####  yield()

yield()方法的作用是放弃当前的CPU资源，让其他任务去占用CPU资源，调用该方法后，线程的状态还是RUNNABLE状态，但是会从RUNNING变成READY。



### Lock类方法

#### Condition类

通过Condition类，我们也可以实现和 Synchronized + wait + notify 一样的Wait/Notify模型，并且还可以实现多路通知功能，也就是在一个Lock对象中可以创建对各Condition实例，线程对象注册在不同的Condition队列中，从而可以有选择性地进行线程通知，在调度线程上更加灵活。同样，与Synchronized一样，在调用Condition对象的方法前，也需要获取Lock锁，否则会出现IllegalMonitorStateException异常。

##### await()

执行该方法后，该线程会进入此condition队列中等待，相当于Object.await()方法。

##### sign()

执行该方法后，会唤醒一个condition队列中的线程，相当于Object.notify()方法。



Code Demo

```Java
public class ConditionUseCase {
    Lock lock = new ReentrantLock();
    Condition condition1 = lock.newCondition();
    Condition condition2 = lock.newCondition();

    public void conditionWait1() throws InterruptedException {
        lock.lock();
        try {
            condition1.await();
        } finally {
            lock.unlock();
        }
        System.out.println("wait1执行完毕");
    }

    public void conditionSignal1() throws InterruptedException {
        lock.lock();

        try {
            condition1.signal();
        }finally {
            lock.unlock();
        }
        System.out.println("sign1执行完毕");
    }
    public void conditionWait2() throws InterruptedException {
        lock.lock();
        try {
            condition2.await();
        } finally {
            lock.unlock();
        }
        System.out.println("wait2执行完毕");
    }

    public void conditionSignal2() throws InterruptedException {
        lock.lock();

        try {
            condition2.signal();
        }finally {
            lock.unlock();
        }
        System.out.println("sign2执行完毕");
    }

}

```

此处condition1和condition2之间的操作就是独立的，condition2的sign()方法不会对condition1中的线程产生影响，同样，condition1也不会对condition2造成影响。

让我们来简单看一下await()方法的简单实现

```Java
//AQS.Condition
public final void await() throws InterruptedException {
    		//判断该线程是否被中断，如果被中断了，就抛出InterruptedException结束线程
            if (Thread.interrupted())
                throw new InterruptedException();
    	    //添加到Condition队列
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                //调用LockSupport.park方法阻塞线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

```Java
//LockSupport的park方法调用了UNSAFE类的park方法阻塞线程
public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
}
```

### LockSupport工具类

上文中发现源码中多次出现了LockSupport这个工具类，当需要阻塞和唤醒一个线程的时候，就会通过LockSupport()来完成相应的功能。LockSupport是JDK提供的一个并发工具类，它定义了一组公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也成为了构建同步组件的基础工具

它在JDK并发包中用到了多次，比如AQS中 Condition类的 await() 方法

<img src="http://img.jjjzzzqqq.top/image-20220329221454478.png" alt="image-20220329221454478" style="zoom:67%;" />

#### park(Object blocker)

阻塞当前线程，blocker是用来标识当前线程阻塞的对象，该对象主要用于问题排查和系统监控。

#### parkNanos(Object blocker, long nanos)

阻塞当前线程，最长不超过nanos纳秒。

#### unpark(Thread thread)

唤醒处于阻塞状态的线程thread

### 阻塞队列

阻塞队列是一个支持阻塞的插入和移除元素的队列。

* 支持阻塞的插入方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满
* 支持阻塞的移除方法：意思是当队列为空时，获取元素的线程会等待队列变为非空

在阻塞队列不可用时，阻塞队列提供了四种不同的处理方式。

![image-20220330161439532](http://img.jjjzzzqqq.top/image-20220330161439532.png)

这里稍微介绍一下几个常用的方法。

#### take() 和 poll()

这两个方法都能从阻塞队列中获取元素，但是take会一直阻塞，而poll会在阻塞队列为空时返回null。

通过这样的特性，我们可以实现不同的线程针对阻塞队列有不同的操作，比如线程池中从任务队列中获取任务。

<img src="http://img.jjjzzzqqq.top/image-20220330161822073.png" alt="image-20220330161822073" style="zoom:67%;" />

```Java
		  boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		  try {
               //timed表示此时该线程是核心线程还是临时线程
                Runnable r = timed ?
                    //临时线程就调用poll,设定超时时间为keepAliveTime,临时线程存活时间
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
              	    //核心线程就调用take,会一直阻塞的等待阻塞队列不为空然后返回一个任务
                    workQueue.take();
                if (r != null)
                    return r;
              	//设置timedOut为true,这个Worker就会进入之后的销毁逻辑
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
```

#### put() 和 offer()

put和offer都会往阻塞队列里添加任务，但是put会阻塞，offer不会阻塞。

#### Java里的阻塞队列

* ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列
* LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列
* PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列
* DelayQueue：一个使用优先级队列实现的无界阻塞队列
* SynchronousQueue：一个不存储元素的阻塞队列
* LinkedTransferQueue：一个由链表结构组成的无界阻塞队列
* LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列

由于笔者对于阻塞队列了解的不是很深入，在此处便不再赘述，之后我会在练习篇给读者手写一个最为常用的阻塞队列ArrayBlockingQueue

### Fork/Join

Fork/Join框架是Java7提供的一个用于进行**并行执行任务**的框架，是一个把大任务分割成若干个小任务，最终汇总每个小人物结果后得到大任务结果的框架。

#### 工作窃取算法

工作窃取算法是指某个线程从其他队列里窃取任务来执行。它可以提高程序并行执行的效率，由于各个工作线程的效率不同，可能会出现A线程已经干完了自己的活，B线程还有很多活没干的情况。如果此时A线程不工作了的话，那么A线程的工作能力就浪费了，于是A线程会窃取B线程的任务，这样就保证了工作的效率。但是因为有多个线程同时访问一个任务队列，为了避免冲突，于是可以使用双端队列，被窃取任务的线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

大体思路：

* 每个线程都有自己的一个WorkQueue，该工作队列是一个**双端队列**。
* 队列支持三个功能push、pop、poll
* push和poll针对队列尾部，pop针对队列头部
* push/pop只能被队列的所有者线程调用，而poll可以被其他线程调用。
* 划分的**子任务调用fork时**，都会被**push**到自己的队列中。
* 默认情况下，工作线程从自己的**双端队列头部**执行pop方法获出任务并执行。
* 当自己的队列为空时，线程随机从**另一个线程的队列末尾**调用poll方法窃取任务。

#### 使用

Fork/Join使用两个类来完成任务的分割以及结果的合并。

* ForkJoinTask：它提供在任务中执行fork() 和 join() 操作的机制。通常情况下，我们不需要直接继承ForkJoinTask类，只需要继承它的子类。
    * RecursiveAction：用于没有返回结果的任务
    * RecursiveTask：用于有返回结果的任务
* ForkJoinPool：ForkJoinTask需要通过ForkJoinPool来执行



Code Demo

```Java
import java.util.concurrent.*;

/**
 * 需求：使用Java多线程计算 1+2+3+4的结果
 */
public class CountTask extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 2;//阈值

    private int start;

    private int end;

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;

        //如果任务足够小就计算任务 fork 边界
        boolean canCompute = (end - start) <= THRESHOLD;
        if(canCompute) {
            for(int i = start; i <= end ;i++) {
                sum+=i;
            }
        } else {
            //任务 fork
            int mid = (start + end) / 2;
            CountTask leftTask = new CountTask(start,mid);
            CountTask rightTask = new CountTask(mid + 1, end);
            //执行子任务
            leftTask.fork();
            rightTask.fork();

            int leftRes = leftTask.join();
            int rightRes = rightTask.join();

            sum = leftRes + rightRes;
        }
        return sum;
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CountTask task = new CountTask(1,4);
		//通过ForkJoinPool来执行ForkJoinTask
        Future<Integer> result = forkJoinPool.submit(task);
        Throwable exception = task.getException();
        System.out.println(exception);
        try {
            System.out.println(result.get());
        } catch (InterruptedException e) {
        } catch (ExecutionException e) {
        }

    }
}

```

### 原子类

原子类又被称为轻量级的线程安全实现机制，通过原子类的API，我们能避免多线程情况下的三大并发问题发生，分别是有序性，可见性，原子性，并且原子类底层并没有使用锁机制来实现，而是通过Unsafe类的CAS方法，实现的乐观锁机制，原子类保证保证原子类的值被更新都是**线程安全**的，不会出现并发问题。

比如AtomicInteger类的API，通过这些API操作对象，就可以保证多线程的线程安全

```Java
public final int get()：获取当前的值
public final int getAndSet(int newValue)：获取当前的值，并设置新的值
public final int getAndIncrement()：获取当前的值，并自增
public final int getAndDecrement()：获取当前的值，并自减
public final int getAndAdd(int delta)：获取当前的值，并加上预期的值
void lazySet(int newValue): 最终会设置成newValue,使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

比如如果想在单机实现一个点赞，秒杀系统，我们就可以利用原子类的特性，通过操作原子类来添加点赞（扣减库存）。

那我们来看一下原子类底层是如何实现的

```Java
    //AtomicInteger.class
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
    public final int decrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
    }
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }

```

我们可以发现，AtomicInteger的方法基本上都是通过unsafe类的方法实现的，那让我们接着看一下unsafe类的代码

```java
//Unsafe.class
public final int getAndAddInt(Object paramObject, long paramLong, int paramInt)
  {
    int i;
    do
      i = getIntVolatile(paramObject, paramLong);
    while (!compareAndSwapInt(paramObject, paramLong, i, i + paramInt));
    return i;
  }

  public final long getAndAddLong(Object paramObject, long paramLong1, long paramLong2)
  {
    long l;
    do
      l = getLongVolatile(paramObject, paramLong1);
    while (!compareAndSwapLong(paramObject, paramLong1, l, l + paramLong2));
    return l;
  }

  public final int getAndSetInt(Object paramObject, long paramLong, int paramInt)
  {
    int i;
    do
      i = getIntVolatile(paramObject, paramLong);
    while (!compareAndSwapInt(paramObject, paramLong, i, paramInt));
    return i;
  }

  public final long getAndSetLong(Object paramObject, long paramLong1, long paramLong2)
  {
    long l;
    do
      l = getLongVolatile(paramObject, paramLong1);
    while (!compareAndSwapLong(paramObject, paramLong1, l, paramLong2));
    return l;
  }

  public final Object getAndSetObject(Object paramObject1, long paramLong, Object paramObject2)
  {
    Object localObject;
    do
      localObject = getObjectVolatile(paramObject1, paramLong);
    while (!compareAndSwapObject(paramObject1, paramLong, localObject, paramObject2));
    return localObject;
  }

```

从源码中发现，内部使用**自旋**的方式进行**CAS更新**(while循环进行CAS更新，如果更新失败，则循环再次重试)。

又从Unsafe类中发现，CAS操作其实只支持下面三个方法。

```Java
public final native boolean compareAndSwapObject(Object paramObject1, long paramLong, Object paramObject2, Object paramObject3);

public final native boolean compareAndSwapInt(Object paramObject, long paramLong, int paramInt1, int paramInt2);

public final native boolean compareAndSwapLong(Object paramObject, long paramLong1, long paramLong2, long paramLong3);

```

分别是三个Native方法，针对Object，Int，Long三种类型变量的操作。

不妨再看看Unsafe的compareAndSwap方法来实现CAS操作，它是一个本地方法，实现位于unsafe.cpp中。

```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END

```

可以看到它通过 `Atomic::cmpxchg` 来实现比较和替换操作。其中参数x是即将更新的值，参数e是原内存的值。

如果是Linux的x86，`Atomic::cmpxchg`方法的实现如下：

```c++
inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}

```

所以说Unsafe类的compareAndSwap方法就是通过Linux的一条汇编指令 cmpxchg原子指令来实现的。

### 等待多线程完成的CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成操作。

比如我们有一个需求，需要解析一个Excel里多个sheet的数据，此时完全可以用多线程执行操作，每个线程解析一个sheet里的数据，等到所有的sheet都解析完之后，程序需要提示解析完成。在这个需求中，要实现主线程等待所有线程完成sheet的解析操作，我们就可以使用CountDownLatch让主线程等待其他线程执行完成。

```Java
/**
 * 需求：实现一个线程等待多个线程执行完成之后再执行
 * 方法1：使用Thread类的Join方法
 * 方法2：使用CountDownLatch类
 */
public class CountDownLatchTest {
    private static final int N = 2;

    static CountDownLatch c = new CountDownLatch(N);

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            System.out.println(1);
            c.countDown();
        }).start();
        new Thread(() -> {
            System.out.println(2);
            c.countDown();
        }).start();

        c.await();
        System.out.println(3);
    }
}
```

c.await()方法会阻塞当前线程，直到N减到0。由于c.countDown()可以用在任何地方，所以我们也可以在一个线程的多个步骤里执行该方法。此处CountDownLatch是一个静态变量，如果不想用静态变量的话，只需要将CountDownLatch变量通过构造方法传进线程即可。

### 同步屏障CyclicBarrier

让一组线程到达一个屏障时被阻塞，直到所有线程到达屏障的时候，屏障才会开门，所有被屏障拦截的线程才会继续运行。

```Java
public class CyclicBarrierTest {
    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) {
        new Thread(() -> {
            try {
                c.await();
            } catch (InterruptedException e) {
            } catch (BrokenBarrierException e) {
            }
            System.out.println(1);
        }).start();

        try {
            c.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println(2);


    }
}
```

我们可以比较一下CountDownLatch和CyclicBarrier的区别，

* 它们内部都有一个N，不同的是CountDownLatch不需要阻塞一个线程就可以让N减一，但是CyclicBarrier必须要一个线程调用await方法阻塞后N才能减一。
* CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置计数器，所以CyclicBarrier适合更加复杂的场景。

CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景，例如，用一个Excel保存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水。

### Semaphore

Semaphore信号量是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

学过操作系统的信号量的话，就能很轻松的明白信号量的含义了

```Java
/**
 * 需求：限制只有N个线程可以获取资源，其他的线程会阻塞
 */
public class SemaphoreTest {
    private static final int THREAD_COUNT = 30;

    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for(int i = 0 ; i < THREAD_COUNT ;i++) {
            threadPool.execute( () -> {
                try {
                    s.acquire();
                    System.out.println("save Date");
                    s.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            });
        }

        threadPool.shutdown();
    }
}

```

acquire方法类似操作系统信号量的P操作

release方法类似操作系统信号量的V操作

### 线程间交换数据的Exchanger

Exchanger是一个用来线程间协作的工具类。Exchanger用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法，当**两个线程都到达同步点**时，这两个线程就可以交换数据，将本线程生产出来的数据**传递给对方**。

```Java
/**
 * 场景：线程间交换数据
 * 需求：校对两个人的工作结果是否一致
 */
public class ExchangerTest {
    //交换器
    private static final Exchanger<String> EXCHANGER = new Exchanger<String>();
    //线程池
    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(() -> {
            String A = "银行流水A";
            try {
                //到达同步点
                String B = EXCHANGER.exchange(A);
                System.out.println(B);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        threadPool.execute(() -> {
            String B = "银行流水B";
            try {
                //到达同步点
                String A = EXCHANGER.exchange(B);
                System.out.println(A);

            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        threadPool.shutdown();
    }

}
```

如果两个线程有一个没有执行exchange方法，则会一直等待，如果担心特殊情况发生，可以使用

```Java
exchange(V x, long timeout, TimeUnit unit)
```

## 练习

### [1114. 按序打印](https://leetcode-cn.com/problems/print-in-order/)

给你一个类：

```Java
public class Foo {
  public void first() { print("first"); }
  public void second() { print("second"); }
  public void third() { print("third"); }
}
```

三个不同的线程 A、B、C 将会共用一个 Foo 实例。

线程 A 将会调用 first() 方法
线程 B 将会调用 second() 方法
线程 C 将会调用 third() 方法
请设计修改程序，以确保 second() 方法在 first() 方法之后被执行，third() 方法在 second() 方法之后被执行。

提示：

尽管输入中的数字似乎暗示了顺序，但是我们并不保证线程在操作系统中的调度顺序。
你看到的输入格式主要是为了确保测试的全面性。


示例 1：

输入：nums = [1,2,3]
输出："firstsecondthird"
解释：
有三个线程会被异步启动。输入 [1,2,3] 表示线程 A 将会调用 first() 方法，线程 B 将会调用 second() 方法，线程 C 将会调用 third() 方法。正确的输出是 "firstsecondthird"。
示例 2：

输入：nums = [1,3,2]
输出："firstsecondthird"
解释：
输入 [1,3,2] 表示线程 A 将会调用 first() 方法，线程 B 将会调用 third() 方法，线程 C 将会调用 second() 方法。正确的输出是 "firstsecondthird"。

题解：

```Java
 /* 
 * 通过信号量实现
 * 针对second 和 third 都设置一个信号量资源，从而实现线程等待的情况
 */
class Foo {
    Semaphore semaphoreTwo ;
    Semaphore semaphoreThree;
    public Foo() {
        semaphoreTwo = new Semaphore(0);
        semaphoreThree = new Semaphore(0);
    }

    public void first(Runnable printFirst) throws InterruptedException {

        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        semaphoreTwo.release();
    }

    public void second(Runnable printSecond) throws InterruptedException {
        semaphoreTwo.acquire();
        // printSecond.run() outputs "second". Do not change or remove this line.
        printSecond.run();
        semaphoreThree.release();
    }

    public void third(Runnable printThird) throws InterruptedException {
        semaphoreThree.acquire();
        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();
    }
}
```

```Java
/**
 * 通过Synchronized实现
 * 通过一个变量的值来实现
 * 类似生产者消费者模型
 */
class Foo {
    int x = 1;
    public Foo() {

    }

    public void first(Runnable printFirst) throws InterruptedException {
        synchronized (Foo.class) {
            // printFirst.run() outputs "first". Do not change or remove this line.
            printFirst.run();
            x = 2;
            Foo.class.notifyAll();
        }
    }

    public void second(Runnable printSecond) throws InterruptedException {
        synchronized (Foo.class) {
            while (x != 2) {
                Foo.class.wait();
            }
            // printSecond.run() outputs "second". Do not change or remove this line.
            printSecond.run();
            x = 3;
            Foo.class.notifyAll();
        }
    }

    public void third(Runnable printThird) throws InterruptedException {
        synchronized (Foo.class) {
            // printThird.run() outputs "third". Do not change or remove this line.
            while (x != 3) {
                Foo.class.wait();
            }
            printThird.run();
        }
    }
}

```



### [1115. 交替打印 FooBar](https://leetcode-cn.com/problems/print-foobar-alternately/)

给你一个类：

```Java
class FooBar {
  public void foo() {
    for (int i = 0; i < n; i++) {
      print("foo");
    }
  }

  public void bar() {
    for (int i = 0; i < n; i++) {
      print("bar");
    }
  }
}
```

两个不同的线程将会共用一个 FooBar 实例：

* 线程 A 将会调用 foo() 方法
* 线程 B 将会调用 bar() 方法

请设计修改程序，以确保 "foobar" 被输出 n 次。

示例 1：

输入：n = 1
输出："foobar"
解释：这里有两个线程被异步启动。其中一个调用 foo() 方法, 另一个调用 bar() 方法，"foobar" 将被输出一次。
示例 2：

输入：n = 2
输出："foobarfoobar"
解释："foobar" 将被输出两次。

分析：典型的盘子容量为1的生产者消费者模型，

* 可以通过synchronized等待唤醒大法实现
* 可以通过Semaphore信号量实现

题解：

```Java
//synchronized 等待唤醒大法
class FooBar {
    private int n;
    //顺序为1，foo输出
    //顺序为2，bar输出
    int shunxu = 1;
    Object lock = new Object();
    public FooBar(int n) {
        this.n = n;
    }

    public void foo(Runnable printFoo) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            synchronized (lock) {
                // printFoo.run() outputs "foo". Do not change or remove this line.
                while (shunxu != 1) lock.wait();
                printFoo.run();
                shunxu = 2;
                lock.notifyAll();
            }
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            synchronized (lock) {
                // printBar.run() outputs "bar". Do not change or remove this line.
                while (shunxu != 2) lock.wait();
                printBar.run();
                shunxu = 1;
                lock.notifyAll();
            }
        }
    }
}
```

```Java
//Semaphore信号量实现
class FooBar {
    private int n;

    Semaphore semaphore1 = new Semaphore(1);
    Semaphore semaphore2 = new Semaphore(0);
    public FooBar(int n) {
        this.n = n;
    }

    public void foo(Runnable printFoo) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            semaphore1.acquire();
            // printFoo.run() outputs "foo". Do not change or remove this line.
            printFoo.run();
            semaphore2.release();
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            semaphore2.acquire();
            // printBar.run() outputs "bar". Do not change or remove this line.
            printBar.run();
            semaphore1.release();
        }
    }
}
```

### 手写生产者消费者模式——多生产者多消费者

注意：写生产者消费者模式有两个注意点

* 线程阻塞的判断条件尽量用While判断（防止多生产者多消费者的时候错误唤醒）
* 使用两个Condition队列代表生产者阻塞线程队列和消费者阻塞线程队列

```Java
//MyService.class
//业务类
public class MyService {
    private Lock lock = new ReentrantLock();
    private Condition producerCondition = lock.newCondition();
    private Condition consumerCondition = lock.newCondition();
    private boolean hasValue =  false;

    public void set() {
        try {
            lock.lock();
            while (hasValue == true) {
                producerCondition.await();
            }
            System.out.println("生产者生产了★");
            hasValue = true;
            consumerCondition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void get() {
        try {
            lock.lock();
            while (hasValue == false) {
                consumerCondition.await();
            }
            System.out.println("消费者消费了☆");
            hasValue = false;
            producerCondition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }


}
```

```Java
//Run.class
//启动10个生产和消费者线程
public class Run {
    public static void main(String[] args) throws InterruptedException {
        MyService service = new MyService();
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                while (true)
                service.set();
            }).start();

            new Thread(() -> {
                while (true)
                service.get();
            }).start();
        }
    }
}
```

运行结果

<img src="http://img.jjjzzzqqq.top/image-20220408221238028.png" alt="image-20220408221238028" style="zoom: 67%;" />

### 实现一个简单的BlockingQueue

思路：我们知道，阻塞队列就是一个天然的生产者消费者模型，那我们就可以用相同的思路来实现。

take方法：如果队列已经空了，那就阻塞线程，取出来一个对象之后就唤醒所有线程

put方法：如果队列已经满了，那就阻塞线程，添加进去一个对象之后就唤醒所有线程

注意：通过数组实现队列，还隐含了一个用数组实现循环队列的思路

```java
/**
 * take方法：如果队列已经空了，那就阻塞线程，取出来一个对象之后就唤醒所有线程
 *
 * put方法：如果队列已经满了，那就阻塞线程，添加进去一个对象之后就唤醒所有线程
 *
 * 注意：通过数组实现队列，还隐含了一个用数组实现循环队列的思路
 */
public class BoundedQueue <T>{

    private Object [] items;

    private Lock lock = new ReentrantLock();

    private int capacity;

    private int size = 0;

    /**
     * 用数组实现的队列，需要保存队头和队尾的下标
     */
    private int head , tail;

    /**
     * 由于队列满了阻塞的生产者线程
     */
    private Condition fullCondition = lock.newCondition();

    /**
     * 由于队列空了阻塞的消费者线程
     */
    private Condition emptyCondition = lock.newCondition();


    public BoundedQueue(int capacity) {
        this.capacity = capacity;
        items = new Object[capacity];
    }

    public void put(T t) {
        lock.lock();
        try {
            while (size == capacity) {
                fullCondition.await();
            }
            items[head] = t;
            //循环添加到数组中
            if(++head == capacity) {
                head = 0;
            }
            ++size;
            emptyCondition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() {
        lock.lock();
        try {
            while (size == 0) {
                emptyCondition.await();
            }
            Object x = items[tail];
            if(++tail == capacity) {
                tail = 0;
            }
            --size;
            fullCondition.signalAll();
            return (T) x;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return null;
    }
}
```

单元测试（希望广大程序员养成单元测试的好习惯）

```Java
public class BoundedQueueTest {
    @Test
    public void putTest() {
        BoundedQueue<Integer> queue = new BoundedQueue<>(1);
        queue.put(1);
        System.out.println("1 put success");
        queue.put(2);
        System.out.println("2 put success");
    }

    @Test
    public void takeTest() {
        BoundedQueue<Integer> queue = new BoundedQueue<>(1);
        queue.put(1);
        System.out.println("1 put success");

        queue.take();
        System.out.println("1 take success");

        queue.take();
        System.out.println("take success");
    }
}

```

执行结果

![image-20220408212224381](http://img.jjjzzzqqq.top/image-20220408212224381.png)

![image-20220408212155174](http://img.jjjzzzqqq.top/image-20220408212155174.png)

可以发现，阻塞队列的API成功阻塞了。

### 循环顺序打印ABC

题目：使用三个线程，A线程打印A，B线程打印B，C线程打印C，循环打印ABC十次。

```Java
/**
 * 顺序打印ABC
 * 循环十次
 */
public class ABC {
    private final static int N = 10;

    /**
     * 1:A执行
     * 2:B执行
     * 3:C执行
     */
    private int shunxun = 1;

    private Object lock = new Object();

    class A extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < N; i++) {
                    while (shunxun != 1) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println("A");
                    shunxun = 2;
                    lock.notifyAll();
                }
            }
            System.out.println("A执行完");
        }
    }

    class B extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < N; i++) {
                    while (shunxun != 2) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println("B");
                    shunxun = 3;
                    lock.notifyAll();
                }
            }
            System.out.println("B执行完");
        }
    }

    class C extends Thread {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < N; i++) {
                    while (shunxun != 3) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println("C");
                    shunxun = 1;
                    lock.notifyAll();
                }
            }
            System.out.println("C执行完");
        }
    }

    @Test
    public void test() throws InterruptedException {
        new A().start();
        new B().start();
        new C().start();
        Thread.sleep(1000);
    }
}

```

输出结果

```java
A
B
C
... × 10
A
B
A执行完
B执行完
C
C执行完
```

