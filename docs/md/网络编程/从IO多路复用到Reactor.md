# 从IO多路复用到Reactor

## 演进

如果要让服务器服务多个客户端，那么最直接的方式就是为**每一条连接创建线程**。

其实创建进程也是可以的，原理是一样的，进程和线程的区别在于线程比较轻量级些，线程的创建和线程间切换的成本要小些，为了描述简述，后面都以线程为例。

处理完业务逻辑后，随着连接关闭后线程也同样要销毁了，但是这样不停地创建和销毁线程，不仅会带来性能开销，也会造成浪费资源，而且如果要连接几万条连接，创建几万个线程去应对也是不现实的。

要这么解决这个问题呢？我们可以使用「资源复用」的方式。

也就是不用再为每个连接创建线程，而是创建一个「**线程池**」，将连接分配给线程，然后一个线程可以处理多个连接的业务。

不过，这样又引来一个新的问题，线程怎样才能高效地处理多个连接的业务？

当一个连接对应一个线程时，线程一般采用「**read -> 业务处理 -> send**」的处理流程，如果当前连接没有数据可读，那么线程会阻塞在 `read` 操作上（ socket 默认情况是阻塞 I/O），不过这种阻塞方式并不影响其他线程。

但是引入了线程池，那么**一个线程要处理多个连接**的业务，线程在处理某个连接的 `read` 操作时，如果遇到没有数据可读，就会发生阻塞，那么线程就没办法继续处理其他连接的业务。

要解决这一个问题，最简单的方式就是将 socket 改成**非阻塞**，然后线程不断地**轮询调用 `read`** 操作来判断是否有数据，这种方式虽然该能够解决阻塞的问题，但是解决的方式比较粗暴，因为轮询是要消耗 CPU 的，而且随着一个 线程处理的连接越多，轮询的效率就会越低。

上面的问题在于，线程并不知道当前连接是否有数据可读，从而需要每次通过 `read` 去试探，试探的过程就会浪费线程资源。

那有没有办法在只有当连接上有数据的时候，线程才去发起读请求呢？答案是有的，实现这一技术的就是 **I/O 多路复用**。

I/O 多路复用技术会用一个系统调用函数来监听我们所有关心的连接，也就说可以在**一个监控线程里面监控很多的连接**。

<img src="https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E5%8D%95Reactor%E5%8D%95%E8%BF%9B%E7%A8%8B.png" alt="image-20220407140028674" style="zoom:67%;" />

**select/poll/epoll 是如何获取网络事件的呢？**

在获取事件时，先把我们要关心的连接传给内核，再由内核检测：

- 如果没有事件发生，线程只需阻塞在这个系统调用，而无需像前面的线程池方案那样轮训调用 read 操作来判断是否有数据。
- 如果有事件发生，内核会**返回产生了事件的连接**，线程就会从阻塞状态返回，然后在用户态中再**处理这些连接对应的业务**即可。

Java NIO就是基于IO多路复用实现的，其底层基于操作系统系统调用情况如下。

| IO模型 | 相对性能 | 操作系统      | Java支持情况                                                 |
| ------ | -------- | ------------- | ------------------------------------------------------------ |
| select | 较高     | windows/Linux | Linux操作系统的kernels2.4内核版本之前，默认使用select；Windows的同步IO都是使用select |
| poll   | 较高     | Linux         | Linux操作系统的kernels2.6内核版本之前，默认使用poll          |
| epoll  | 高       | Linux         | Linux操作系统的kernels2.6内核版本及以后，默认使用epoll       |



## Reactor

那么当前开源软件能做到网络高性能的原因是因为IO多路复用吗？

是的，基本是基于 I/O 多路复用，用过 I/O 多路复用接口写网络程序的同学，肯定知道是**面向过程**的方式写代码的，这样的开发的效率不高。

于是，大佬们基于**面向对象**的思想，对 I/O 多路复用作了一层封装，让使用者不用考虑底层网络 API 的细节，只需要关注应用代码的编写。

大佬们还为这种模式取了个让人第一时间难以理解的名字：**Reactor 模式**。

Reactor 翻译过来的意思是「反应堆」，可能大家会联想到物理学里的核反应堆，实际上并不是的这个意思。

这里的反应指的是「**对事件反应**」，也就是**来了一个事件，Reactor 就有相对应的反应/响应**。

事实上，Reactor 模式也叫 `Dispatcher` 模式，我觉得这个名字更贴合该模式的含义，即 **I/O 多路复用监听事件，收到事件后，根据事件类型分配（Dispatch）给某个进程 / 线程**。

Reactor 模式主要由 **Reactor** 和**处理资源池**这两个核心部分组成，它俩负责的事情如下：

- Reactor 负责**监听**和**分发**事件，事件类型包含**连接事件**、**读写事件**，连接事件交给**Acceptor**，读写事件交给**Handler**
- 处理资源池负责处理事件，如 read -> 业务逻辑 -> send；

Reactor 模式是灵活多变的，可以应对不同的业务场景，灵活在于：

- Reactor 的数量可以只有一个，也可以有多个；
- 处理资源池可以是单个进程 / 线程，也可以是多个进程 /线程；

将上面的两个因素排列组设一下，理论上就可以有 4 种方案选择：

- 单 Reactor 单进程 / 线程；
- 单 Reactor 多进程 / 线程；
- 多 Reactor 单进程 / 线程；
- 多 Reactor 多进程 / 线程；

其中，「多 Reactor 单进程 / 线程」实现方案相比「单 Reactor 单进程 / 线程」方案，不仅复杂而且也没有性能优势，因此实际中并没有应用。

剩下的 3 个方案都是比较经典的，且都有应用在实际的项目中：

- **单 Reactor 单进程 / 线程**：Redis
- **单 Reactor 多线程 / 进程**：Kafka
- **多 Reactor 多进程 / 线程**：Nginx、Memcache

接下来，分别介绍这三个经典的 Reactor 方案。

一般来说，C 语言实现的是「**单 Reactor 单进程**」的方案，因为 C 语编写完的程序，运行后就是一个独立的进程，不需要在进程中再创建线程。

而 Java 语言实现的是「**单 Reactor 单线程**」的方案，因为 Java 程序是跑在 Java 虚拟机这个进程上面的，虚拟机中有很多线程，我们写的 Java 程序只是其中的一个线程而已。

## 单 Reactor 单进程

我们来看看「**单 Reactor 单进程**」的方案示意图：

<img src="https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E5%8D%95Reactor%E5%8D%95%E8%BF%9B%E7%A8%8B.png" alt="img" style="zoom: 50%;" />

可以看到进程里有**Reactor**、**Acceptor**、**Handler**这三个对象

* Reactor对象的作用是监听和分发事件
* Acceptor对象的作用是获取连接
* Handler对象的作用是处理业务

对象里的 select、accept、read、send 是系统调用函数，dispatch 和 「业务处理」是需要完成的操作，其中 dispatch 是分发事件操作。

接下来，介绍下「单 Reactor 单进程」这个方案：

- Reactor 对象通过 select （IO 多路复用接口） 监听事件，收到事件后通过 dispatch 进行分发，具体分发给 Acceptor 对象还是 Handler 对象，还要看收到的事件类型；
- 如果是连接建立的事件，则交由 Acceptor 对象进行处理，Acceptor 对象会通过 accept 方法 获取连接，并创建一个 Handler 对象来处理后续的响应事件；
- 如果不是连接建立事件， 则交由当前连接对应的 Handler 对象来进行响应；
- Handler 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程。

单 Reactor 单进程的方案因为全部工作都在同一个进程内完成，所以实现起来比较简单，不需要考虑进程间通信，也不用担心多进程竞争。

但是，这种方案存在 2 个缺点：

1. 因为只有一个进程/线程，**无法充分利用多核CPU的性能**；
2. Handler对象在进行业务处理时，由于是单进程的，这一个时刻它无法处理其他连接的事件，**如果业务处理耗时比较长，那么就会造成响应的延迟**；

所以，单 Reactor 单进程的方案**不适用计算机密集型的场景，只适用于业务处理非常快速的场景**。

Redis 是由 C 语言实现的，它采用的正是「单 Reactor 单进程」的方案，因为 Redis 业务处理主要是在内存中完成，操作的速度是很快的，性能瓶颈不在 CPU 上，所以 Redis 对于命令的处理是单进程的方案

` Talk is cheap. Show me the code `

基于Java NIO实现的朴素的Reactor模型，并没有抽象出Reactor、Acceptor、Handler对象，大家可以用来和之后的Reactor模式进行对比。

```Java
public class NIOServer {

    public static void main(String[] args) throws IOException {
		//初始化selector
        Selector selector = Selector.open();
		//初始化ServerSocketChannel
        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        //设置非阻塞
        ssChannel.configureBlocking(false);
        //注册
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);
		
        //bind port
        ServerSocket serverSocket = ssChannel.socket();
        InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8888);
        serverSocket.bind(address);

        while (true) {

            selector.select();
            //IO多路复用返回的事件
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();

            while (keyIterator.hasNext()) {

                SelectionKey key = keyIterator.next();

                if (key.isAcceptable()) {

                    SocketChannel socketChannel = serverSocket.accept();
                    socketChannel.configureBlocking(false);
                    // 这个新连接主要用于从客户端读取数据
                    socketChannel.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()) {

                    SocketChannel sChannel = (SocketChannel) key.channel();
                    //1. read from sChannel
                    //2. handle data
                    //3. send result
                    sChannel.close();
                }

                keyIterator.remove();
            }
        }
    }
}
```

Reactor模式

```Java
//Reactor
package NIO.单Reactor单线程;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class Reactor  {

    final Selector selector;
    final ServerSocketChannel serverSocket;


    Reactor(int port) throws IOException {
        //Reactor初始化
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        //非阻塞
        serverSocket.configureBlocking(false);

        //注册
        SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        //attach callback object, Acceptor
        sk.attach(new Acceptor());

    }

    public void run() {
        try {
            while (true) {
                selector.select();
                Set<SelectionKey> keySet = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keySet.iterator();
                while (iterator.hasNext()) {
                    //Reactor负责dispatch收到的事件
                    dispatch((SelectionKey) (iterator.next()));
                }
                keySet.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    void dispatch (SelectionKey key) {
        Acceptor acceptor = (Acceptor) key.attachment();
        if(acceptor != null) {
            acceptor.run();
        }
    }

    class Acceptor implements Runnable{

        public void run() {
            try {
                SocketChannel channel = serverSocket.accept();
                if(channel != null) {
                    new Handler(selector, channel).run();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

```

```Java
//Handler
package NIO.单Reactor单线程;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;

public class Handler {
    final SocketChannel channel;
    final SelectionKey selectionKey;
    ByteBuffer input = ByteBuffer.allocate(1024);
    ByteBuffer output = ByteBuffer.allocate(1024);
    static final int READING = 0, SENDING = 1;
    int state = READING;

    /**
     * c是新建立连接的Channel，所以需要初始化一下设置
     */
    Handler(Selector selector, SocketChannel c) throws IOException {
        channel = c;
        c.configureBlocking(false);
        // Optionally try first read now
        //代表这个channel永远不会被Select
        selectionKey = channel.register(selector, 0);

        //将Handler作为callback对象
        selectionKey.attach(this);

        //第二步,注册Read就绪事件
        selectionKey.interestOps(SelectionKey.OP_READ);
        //唤醒selector，因为有可能阻塞住了
        selector.wakeup();
    }

    /**
     * handle method
     */
    public void run() {
        try {
            if (state == READING) {
                read();
            } else if (state == SENDING) {
                send();
            }
        } catch (IOException ex) { /* ... */ }
    }

    void read() throws IOException {
        channel.read(input);
        if (inputIsComplete()) {

            process();

            state = SENDING;
            // Normally also do first write now

            //第三步,接收write就绪事件
            selectionKey.interestOps(SelectionKey.OP_WRITE);
        }
    }

    void process() {
        /* ... */
        return;
    }

    void send() throws IOException {
        channel.write(output);

        //write完就结束了, 关闭select key
        if (outputIsComplete()) {
            selectionKey.cancel();
        }
    }

    boolean inputIsComplete() {
        /* ... */
        return false;
    }

    boolean outputIsComplete() {

        /* ... */
        return false;
    }

}

```



## 单 Reactor 多进程

如果要克服「单 Reactor 单线程 / 进程」方案的缺点，那么就需要引入多线程 / 多进程来进行业务处理，这样就产生了**单 Reactor 多线程 / 多进程**的方案。

闻其名不如看其图，先来看看「单 Reactor 多线程」方案的示意图如下：

<img src="https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E5%8D%95Reactor%E5%A4%9A%E7%BA%BF%E7%A8%8B.png" style="zoom: 50%;" />

详细说一下这个方案：

- Reactor 对象通过 select （IO 多路复用接口） 监听事件，收到事件后通过 dispatch 进行分发，具体分发给 Acceptor 对象还是 Handler 对象，还要看收到的事件类型；
- 如果是连接建立的事件，则交由 Acceptor 对象进行处理，Acceptor 对象会通过 accept 方法 获取连接，并创建一个 Handler 对象来处理后续的响应事件；
- 如果不是连接建立事件， 则交由当前连接对应的 Handler 对象来进行响应；

上面的三个步骤和单 Reactor 单线程方案是一样的，接下来的步骤就开始不一样了：

- Handler 对象不再负责业务处理，**只负责数据的接收和发送**，Handler 对象通过 read 读取到数据后，会将数据发给子线程里的 **Processor 对象**进行业务处理；
- 子线程里的 Processor 对象就进行业务处理，处理完后，**将结果发给主线程中的 Handler 对象**，接着由 Handler 通过 send 方法将响应结果发送给 client；

单 Reator 多线程的方案优势在于**能够充分利用多核 CPU 的能**，那既然引入多线程，那么自然就带来了**多线程竞争资源**的问题。

例如，子线程完成业务处理后，要把结果传递给Handler 进行发送，这里涉及共享数据的竞争。（一个Handler可能有多个Processor 对象）

要避免多线程由于竞争共享资源而导致数据错乱的问题，就需要在**操作共享资源前加上互斥锁**，以保证任意时间里只有一个线程在操作共享资源，待该线程操作完释放互斥锁后，其他线程才有机会操作共享数据。

`Show me the code`

Handler类（Reactor类不变）

```Java
public class Handler {
    final SocketChannel channel;
    final SelectionKey selectionKey;
    ByteBuffer input = ByteBuffer.allocate(1024);
    ByteBuffer output = ByteBuffer.allocate(1024);
    static final int READING = 0, SENDING = 1;
    int state = READING;

    /**
     * 每个Handler里有一个线程池
     */
    ExecutorService pool = Executors.newFixedThreadPool(2);


    static final int PROCESSING = 3;

    Handler(Selector selector, SocketChannel c) throws IOException {
        channel = c;
        c.configureBlocking(false);
        // Optionally try first read now
        selectionKey = channel.register(selector, 0);

        //将Handler作为callback对象
        selectionKey.attach(this);

        //第二步,注册Read就绪事件
        selectionKey.interestOps(SelectionKey.OP_READ);
        selector.wakeup();
    }

    boolean inputIsComplete() {
        /* ... */
        return false;
    }

    boolean outputIsComplete() {

        /* ... */
        return false;
    }

    void process() {
        /* ... */
        return;
    }

    public void run() {
        try {
            if (state == READING) {
                read();
            } else if (state == SENDING) {
                send();
            }
        } catch (IOException ex) { /* ... */ }
    }


    synchronized void read() throws IOException {
        // ...
        channel.read(input);
        if (inputIsComplete()) {
            state = PROCESSING;
            //使用线程pool异步执行
            pool.execute(new Processer());
        }
    }

    void send() throws IOException {
        channel.write(output);

        //write完就结束了, 关闭select key
        if (outputIsComplete()) {
            selectionKey.cancel();
        }
    }

    synchronized void processAndHandOff() {
        process();
        state = SENDING;
        // or rebind attachment
        //process完,开始等待write就绪
        selectionKey.interestOps(SelectionKey.OP_WRITE);
    }

    class Processer implements Runnable {
        public void run() {
            processAndHandOff();
        }
    }

}

```

聊完单 Reactor 多线程的方案，接着来看看单 Reactor 多进程的方案。

事实上，单 Reactor 多进程相比单 Reactor 多线程实现起来很麻烦，主要因为要考虑子进程 <-> 父进程的双向通信，并且父进程还得知道子进程要将数据发送给哪个客户端。

而多线程间可以共享数据，虽然要额外考虑并发问题，但是这远比进程间通信的复杂度低得多，因此实际应用中也看不到单 Reactor 多进程的模式。

另外，「单 Reactor」的模式还有个问题，**因为一个 Reactor 对象承担所有事件的监听和响应，而且只在主线程中运行，在面对瞬间高并发的场景时，容易成为性能的瓶颈的地方**。

## 多 Reactor 多进程

要解决「单 Reactor」的问题，就是将「单 Reactor」实现成「多 Reactor」，这样就产生了第 **多 Reactor 多进程 / 线程**的方案。

老规矩，闻其名不如看其图。多 Reactor 多进程 / 线程方案的示意图如下（以线程为例）：

<img src="https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E4%B8%BB%E4%BB%8EReactor%E5%A4%9A%E7%BA%BF%E7%A8%8B.png" style="zoom: 50%;" />

方案详细说明如下：

- 主线程中的 MainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 对象中的 accept 获取连接，将**新的连接分配给某个子线程**（之前是自己负责dispatch；
- 子线程中的 SubReactor 对象将 MainReactor 对象分配的连接加入 select 继续进行监听，并创建一个 Handler 用于处理连接的响应事件。
- 如果有新的事件发生时，SubReactor 对象会调用当前连接对应的 Handler 对象来进行响应。
- Handler 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程。

多 Reactor 多线程的方案虽然看起来复杂的，但是实际实现时比单 Reactor 多线程的方案要简单的多，原因如下：

- 主线程和子线程分工明确，**主线程只负责接收新连接**，**子线程负责完成后续的业务处理**。
- 主线程和子线程的交互很简单，**主线程只需要把新连接传给子线程**，子线程无须返回数据，直接就可以在子线程将处理结果发送给客户端。

大名鼎鼎的两个开源软件 Netty 和 Memcache 都采用了「多 Reactor 多线程」的方案。

采用了「多 Reactor 多进程」方案的开源软件是 Nginx，不过方案与标准的多 Reactor 多进程有些差异。

具体差异表现在主进程中仅仅用来初始化 socket，并没有创建 mainReactor 来 accept 连接，而是由子进程的 Reactor 来 accept 连接，通过锁来控制一次只有一个子进程进行 accept（防止出现惊群现象），子进程 accept 新连接后就放到自己的 Reactor 进行处理，不会再分配给其他子进程，之后的代码就是类似这种这样实现的，由subReactors来accept连接。

`Show me the code`

Reactor类（Handler不变）

```Java
public class Reactor {

    //subReactors集合, 一个selector代表一个subReactor
    Selector[] selectors=new Selector[2];
    int next = 0;
    final ServerSocketChannel serverSocket;

    Reactor(int port) throws IOException
    { //Reactor初始化
        selectors[0]= Selector.open();
        selectors[1]= Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        //非阻塞
        serverSocket.configureBlocking(false);


        //分步处理,第一步,接收accept事件
        SelectionKey sk =
                serverSocket.register( selectors[0], SelectionKey.OP_ACCEPT);
        //attach callback object, Acceptor
        sk.attach(new Acceptor());
    }

    public void run()
    {
        try
        {
            while (!Thread.interrupted())
            {
                //两个subReactor 轮流处理事件
                for (int i = 0; i <2 ; i++)
                {
                    selectors[i].select();
                    Set selected =  selectors[i].selectedKeys();
                    Iterator it = selected.iterator();
                    while (it.hasNext())
                    {
                        //Reactor负责dispatch收到的事件
                        dispatch((SelectionKey) (it.next()));
                    }
                    selected.clear();
                }

            }
        } catch (IOException ex)
        { /* ... */ }
    }

    void dispatch(SelectionKey k)
    {
        Runnable r = (Runnable) (k.attachment());
        //调用之前注册的callback对象
        if (r != null)
        {
            r.run();
        }
    }


    class Acceptor implements Runnable{ // ...
        public synchronized void run() throws IOException
        {
            SocketChannel connection =
                    serverSocket.accept(); //主selector负责accept
            if (connection != null)
            {
                new Handler(selectors[next], connection); //选个subReactor去负责接收到的connection
            }
            if (++next == selectors.length) next = 0;
        }
    }
}
```

参考文章：

[小林coding](https://xiaolincoding.com/os/8_network_system/reactor.html)

[Java全栈知识体系-NIO](https://www.pdai.tech/md/java/io/java-io-nio-select-epoll.html)

[Java全栈知识体系-IO多路复用详解](https://www.pdai.tech/md/java/io/java-io-nio.html)
