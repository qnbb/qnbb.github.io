---
layout: post
title: Socket 通道
categories: [Java, NIO]
description: Selector
keywords: JAVA, NIO
---



选择器提供选择执行已经就绪的任务的能力，这使得多元 I/O 成为可能。就绪选择和多元执行使得单线程能够有效率地同时管理多个 I/O 通道(channels)。C/C++代码的工具箱中，许多年前就已经有 select( )和 poll( )这两个POSIX(可移植性操作系统接口)系统调用可供使用了。许过操作系统也提供相似的功能，但对Java 程序员来说就绪选择功能直到 JDK 1.4 才成为可行的方案。

继SocketChanne银行的例子。想象一下，一个有三个传送通道的银行。在传统的(非选择器)的场景里，想象一下每个银行的传送通道都有一个气动导管，传送到银行里它对应的出纳员的窗口，并且每一个窗口与其他窗口是用墙壁分隔开的。这意味着每个导管(通道)需要一个专门的出纳员(工作线程)。这种方式不易于扩展，而且也是十分浪费的。对于每个新增加的导管(通道)，都需要一个新的出纳员，以及其他相关的经费，如表格、椅子、纸张的夹子(内存、CPU 周期、上下文切换)等等。并且当事情变慢下来时，这些资源(以及相关的花费)大多数时候是闲置的。

现在想象一下另一个不同的场景，每一个气动导管(通道)都只与一个出纳员的窗口连接。这个窗口有三个槽可以放置运输过来的物品(数据缓冲区)，每个槽都有一个指示器(选择键，selection key)，当运输的物品进入时会亮起。同时想象一下出纳员(工作线程)有一个花尽量多的时间阅读《自己动手编写个人档案》一书的癖好。在每一段的最后，出纳员看一眼指示灯(调用select( )函数)，来决定人一个通道是否已经就绪(就绪选择)。在传送带闲置时，出纳员(工作线程)可以做其他事情，但需要注意的时候又可以进行及时的处理。

### Selectors 选择器基础

从最基础的层面来看，选择器提供了询问通道是否已经准备好执行每个 I/0 操作的能力。例如，我们需要了解一个 SocketChannel 对象是否还有更多的字节需要读取，或者我们需要知道ServerSocketChannel 是否有需要准备接受的连接。

在与 SelectableChannel 联合使用时，选择器提供了这种服务，但这里面有更多的事情需要去了解。**就绪选择的真正价值在于潜在的大量的通道可以同时进行就绪状态的检查。调用者可以轻松地决定多个通道中的哪一个准备好要运行。**

有两种方式可以选择：
1. 激发的线程可以处于休眠状态，直到一个或者多个注册到选择器的通道就绪
2. 或者它也可以周期性地轮询选择器，看看从上次检查之后，是否有通道处于就绪状态。如果您考虑一下需要管理大量并发的连接的网络服务器(webserver)的实现，就可以很容易地想到如何善加利用这些能力。

传统的监控多个 socket 的 Java 解决方案是为每个 socket 创建一个线程并使得线程可以在 read( )调用中阻塞，直到数据可用。这事实上将每个被阻塞的线程当作了 socket 监控器，并将 Java 虚拟机的线程调度当作了通知机制。

正的就绪选择必须由操作系统来做。**操作系统的一项最重要的功能就是处理 I/O 请求并通知各个线程它们的数据已经准备好了。**选择器类提供了这种抽象，使得 Java 代码能够以可移植的方式，请求底层的操作系统提供就绪选择服务。

#### 选择器，可选择通道和选择键类


##### 选择器(Selector)

选择器类管理着一个被注册的通道集合的信息和它们的就绪状态。通道是和选择器一起被注册的，并且使用选择器来更新通道的就绪状态。当这么做的时候，可以选择将被激发的线程挂起，直到有就绪的的通道。

##### 可选择通道(SelectableChannel)
这个抽象类提供了实现通道的可选择性所需要的公共方法。它是所有支持就绪检查的通道类的父类。FileChannel 对象不是可选择的，因为它们没有继承 SelectableChannel。

所有 socket 通道都是可选择的，包括从管道(Pipe)对象的中获得的通道。SelectableChannel 可以被注册到 Selector 对象上，同时可以指定对那个选择器而言，那种操作是感兴趣的。一个通道可以被注册到多个选择器上，但对每个选择器而言只能被注册一次。

通道在被注册到一个选择器上之前，必须先设置为非阻塞模式(通过调用 configureBlocking(false))

##### 选择键(SelectionKey)
选择键封装了特定的通道与特定的选择器的注册关系。选择键对象被SelectableChannel.register( ) 返回并提供一个表示这种注册关系的标记。选择键包含了两个比特集(以整数的形式进行编码)，指示了该注册关系所关心的通道操作，以及通道已经准备好的操作。

就绪选择相关类的关系图：
![](/images/posts/java/selector_channel.jpg)

调用可选择通道的 register( )方法会将它注册到一个选择器上。如果您试图注册一个处于阻塞状态的通道，register( )将抛出未检查的 IllegalBlockingModeException 异常。此外，通道一旦被注册，就不能回到阻塞状态。试图这么做的话，将在调用 configureBlocking( )方法时将抛出IllegalBlockingModeException 异常。

试图注册一个已经关闭的 SelectableChannel 实例的话，也将抛出
ClosedChannelException 异常，就像方法原型指示的那样。

进一步了解 register( )和 SelectableChannel 的其他方法之前，让我们先了解一下Selector 类的 API，以确保我们可以更好地理解这种关系：

```
public abstract class Selector
{
public static Selector open( ) throws IOException
public abstract boolean isOpen( );
public abstract void close( ) throws IOException;
public abstract SelectionProvider provider( );
public abstract int select( ) throws IOException;
public abstract int select (long timeout) throws IOException;
public abstract int selectNow( ) throws IOException;
public abstract void wakeup( );
public abstract Set keys( );
public abstract Set selectedKeys( );
}
```

选择器维护了一个需要监控的通道的集合。一个给定的通道可以被注册到多于一个 的 选 择 器 上 ， 而 且 不 需 要 知 道 它 被 注 册 了 那 个 Selector 对 象 上 。 将 register( ) 放 在SelectableChannel 上而不是 Selector 上，这种做法看起来有点随意。它将返回一个封装了两个对象的关系的选择键对象。

#### 建立选择器
建立监控三个 Socket 通道的选择器：
```
Selector selector = Selector.open( );
channel1.register (selector, SelectionKey.OP_READ);
channel2.register (selector, SelectionKey.OP_WRITE);
channel3.register (selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
// Wait up to 10 seconds for a channel to become ready
readyCount = selector.select(10000);
```

这些代码创建了一个新的选择器，然后将这三个(已经存在的)socket 通道注册到选择器上，而且感兴趣的操作各不相同。select( )方法在将线程置于睡眠状态，直到这些刚兴趣的事情中的操作中的一个发生或者 10 秒钟的时间过去。


### 使用选择键

SelectionKey 类的 API:
```
package java.nio.channels;
public abstract class SelectionKey
{
public static final int OP_READ;
public static final int OP_WRITE;
public static final int OP_CONNECT;
public static final int OP_ACCEPT;
public abstract SelectableChannel channel( );
public abstract Selector selector( );
public abstract void cancel( );
public abstract boolean isValid( );
public abstract int interestOps( );
public abstract void interestOps (int ops);
public abstract int readyOps( );
public final boolean isReadable( )
public final boolean isWritable( )
public final boolean isConnectable( )
public final boolean isAcceptable( )
public final Object attach (Object ob)
public final Object attachment( )
}
```
一个键表示了一个特定的通道对象和一个特定的选择器对象之间的注册关 系 。 您 可 以 看 到 前 两 个 方 法 中 反 映 了 这 种 关 系 。 channel( ) 方 法 返 回 与 该 键 相 关 的SelectableChannel 对象,而 selector( )则返回相关的 Selector 对象。

一个 SelectionKey 对象包含两个以整数形式进行编码的比特掩码:一个用于指示那些通道/选择器组合体所关心的操作**(instrest 集合)**,另一个表示通道准备好要执行的操作**(ready 集合)**。

1. interest 集合:当前channel感兴趣的操作,此类操作将会在下一次选择器select操作时被交付,可以通过selectionKey.interestOps(int)进行修改.
2. ready 集合:表示此选择键上,已经就绪的操作.每次select时,选择器都会对ready集合进行更新;外部程序无法修改此集合.


SelectionKey 类定义了四个便于使用的布尔方法来为您测试这些比特值:isReadable( ),isWritable( ),isConnectable( ), 和 isAcceptable( )。每一个方法都与使用特定掩码来测试 readyOps( )方法的结果的效果相同。例如:
``` 
 (key.isWritable( ))
```
等价于:

``` 
if ((key.readyOps( ) & SelectionKey.OP_WRITE) != 0)
```


### 使用选择器

#### 选择过程
每一个 Selector 对象维护三个键的集合：
```
public abstract class Selector
{
// This is a partial API listing
public abstract Set keys( );
public abstract Set selectedKeys( );
public abstract int select( ) throws IOException;
public abstract int select (long timeout) throws IOException;
public abstract int selectNow( ) throws IOException;
public abstract void wakeup( );
}
```

Selector 类的核心是选择过程。基本上来说,选择器是对 select( )、poll( )等本地调用(native call)或者类似的操作系统特定的系统调用的一个包装。但是 Selector 所作的不仅仅是简单地向本地代码传送参数。它对每个选择操作应用特定的过程。对这个过程的理解，是合理地管理键和它们所表示的状态信息的基础。

#### 通过Selector选择通道

我们知道选择器维护注册过的通道的集合，并且这种注册关系都被封装在SelectionKey当中。接下来我们简单的了解一下Selector维护的三种类型SelectionKey集合：

- 已注册的键的集合(Registered key set)
所有与选择器关联的通道所生成的键的集合称为已经注册的键的集合。并不是所有注册过的键都仍然有效。这个集合通过keys()方法返回，并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引发java.lang.UnsupportedOperationException。

- 已选择的键的集合(Selected key set)
已注册的键的集合的子集。这个集合的每个成员都是相关的通道被选择器(在前一个选择操作中)判断为已经准备好的，并且包含于键的interest集合中的操作。这个集合通过selectedKeys()方法返回(并有可能是空的)。 
不要将已选择的键的集合与ready集合弄混了。这是一个键的集合，每个键都关联一个已经准备好至少一种操作的通道。每个键都有一个内嵌的ready集合，指示了所关联的通道已经准备好的操作。键可以直接从这个集合中移除，但不能添加。试图向已选择的键的集合中添加元素将抛出java.lang.UnsupportedOperationException。

- 已取消的键的集合(Cancelled key set)
已注册的键的集合的子集，这个集合包含了cancel()方法被调用过的键(这个键已经被无效化)，但它们还没有被注销。这个集合是选择器对象的私有成员，因而无法直接访问。

在刚初始化的Selector对象中，这三个集合都是空的。通过Selector的select（）方法可以选择已经准备就绪的通道（这些通道包含你感兴趣的的事件）。比如你对读就绪的通道感兴趣，那么select（）方法就会返回读事件已经就绪的那些通道。下面是Selector几个重载的select()方法：
1. select():阻塞到至少有一个通道在你注册的事件上就绪了。 
2. select(long timeout)：和select()一样，但最长阻塞事件为timeout毫秒。 
3. selectNow():非阻塞，只要有通道就绪就立刻返回。

select()方法返回的int值表示有多少通道已经就绪,是自上次调用select()方法后有多少通道变成就绪状态。之前在select（）调用时进入就绪的通道不会在本次调用中被记入，而在前一次select（）调用进入就绪但现在已经不在处于就绪的通道也不会被记入。例如：

首次调用select()方法，如果有一个通道变成就绪状态，返回了1，若再次调用select()方法，如果另一个通道就绪了，它会再次返回1。如果对第一个就绪的channel没有做任何操作，现在就有两个就绪的通道，但在每次select()方法调用之间，只有一个通道就绪了。

一旦调用select()方法，并且返回值不为0时，则可以通过调用Selector的selectedKeys()方法来访问已选择键集合。如下： 
```
Set selectedKeys=selector.selectedKeys(); 
```

进而可以放到和某SelectionKey关联的Selector和Channel。如下所示：
```
Set selectedKeys = selector.selectedKeys();
Iterator keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.
    } else if (key.isConnectable()) {
        // a connection was established with a remote server.
    } else if (key.isReadable()) {
        // a channel is ready for reading
    } else if (key.isWritable()) {
        // a channel is ready for writing
    }
    keyIterator.remove();
}
```

关于Selector执行选择的过程:

我们知道调用select（）方法进行通道，现在我们再来深入一下选择的过程，也就是select（）执行过程。当select（）被调用时将执行以下几步：

1. 首先检查已取消键集合，也就是通过cancle()取消的键。如果该集合不为空，则清空该集合里的键，同时该集合中每个取消的键也将从已注册键集合和已选择键集合中移除。（一个键被取消时，并不会立刻从集合中移除，而是将该键“拷贝”至已取消键集合中，这种取消策略就是我们常提到的“延迟取消”。）
2. 再次检查已注册键集合（准确说是该集合中每个键的interest集合）。系统底层会依次询问每个已经注册的通道是否准备好选择器所感兴趣的某种操作，一旦发现某个通道已经就绪了，则会首先判断该通道是否已经存在在已选择键集合当中，如果已经存在，则更新该通道在已注册键集合中对应的键的ready集合，如果不存在，则首先清空该通道的对应的键的ready集合，然后重设ready集合，最后将该键存至已注册键集合中。这里需要明白，当更新ready集合时，在上次select（）中已经就绪的操作不会被删除，也就是ready集合中的元素是累积的，比如在第一次的selector对某个通道的read和write操作感兴趣，在第一次执行select（）时，该通道的read操作就绪，此时该通道对应的键中的ready集合存有read元素，在第二次执行select()时，该通道的write操作也就绪了，此时该通道对应的ready集合中将同时有read和write元素。

#### 停止选择

选择器执行选择的过程，系统底层会依次询问每个通道是否已经就绪，这个过程可能会造成调用线程进入阻塞状态,那么我们有以下三种方式可以唤醒在select（）方法中阻塞的线程。

1. 通过调用Selector对象的wakeup（）方法让处在阻塞状态的select()方法立刻返回 
该方法使得选择器上的第一个还没有返回的选择操作立即返回。如果当前没有进行中的选择操作，那么下一次对select()方法的一次调用将立即返回。
2. 通过close（）方法关闭Selector** 
该方法使得任何一个在选择操作中阻塞的线程都被唤醒（类似wakeup（）），同时使得注册到该Selector的所有Channel被注销，所有的键将被取消，但是Channel本身并不会关闭。
3. 调用interrupt() 
调用该方法会使睡眠的线程抛出InterruptException异常，捕获该异常并在调用wakeup()

上面有些人看到“系统底层会依次询问每个通道”时可能在想如果已选择键非常多是，会不会耗时较长？答案是肯定的。但是我想说的是通常你可以选择忽略该过程。

### Selector完整实例

#### Server 端代码
```
public class ServerSocketChannelTest {

    private int size = 1024;
    private ServerSocketChannel socketChannel;
    private ByteBuffer byteBuffer;
    private Selector selector;
    private final int port = 8998;
    private int remoteClientNum=0;

    public ServerSocketChannelTest() {
        try {
            initChannel();
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(-1);
        }
    }

    public void initChannel() throws Exception {
        socketChannel = ServerSocketChannel.open();
        socketChannel.configureBlocking(false);
        socketChannel.bind(new InetSocketAddress(port));
        System.out.println("listener on port:" + port);
        selector = Selector.open();
        socketChannel.register(selector, SelectionKey.OP_ACCEPT);
        byteBuffer = ByteBuffer.allocateDirect(size);
        byteBuffer.order(ByteOrder.BIG_ENDIAN);
    }

    private void listener() throws Exception {
        while (true) {
            int n = selector.select();
            if (n == 0) {
                continue;
            }
            Iterator<SelectionKey> ite = selector.selectedKeys().iterator();
            while (ite.hasNext()) {
                SelectionKey key = ite.next();
                //a connection was accepted by a ServerSocketChannel.
                if (key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel channel = server.accept();
                    registerChannel(selector, channel, SelectionKey.OP_READ);
                    remoteClientNum++;
                    System.out.println("online client num="+remoteClientNum);
                    replyClient(channel);
                }
                //a channel is ready for reading
                if (key.isReadable()) {
                    readDataFromSocket(key);
                }

                ite.remove();//must
            }

        }
    }

    protected void readDataFromSocket(SelectionKey key) throws Exception {
        SocketChannel socketChannel = (SocketChannel) key.channel();
        int count;
        byteBuffer.clear();
        while ((count = socketChannel.read(byteBuffer)) > 0) {
            byteBuffer.flip(); // Make buffer readable
            // Send the data; don't assume it goes all at once
            while (byteBuffer.hasRemaining()) {
                socketChannel.write(byteBuffer);
            }
            byteBuffer.clear(); // Empty buffer
        }
        if (count < 0) {
            socketChannel.close();
        }
    }

    private void replyClient(SocketChannel channel) throws IOException {
        byteBuffer.clear();
        byteBuffer.put("hello client!\r\n".getBytes());
        byteBuffer.flip();
        channel.write(byteBuffer);
    }

    private void registerChannel(Selector selector, SocketChannel channel, int ops) throws Exception {
        if (channel == null) {
            return;
        }
        channel.configureBlocking(false);
        channel.register(selector, ops);
    }


    public static void main(String[] args) {
        try {
            new ServerSocketChannelTest().listener();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### Client 端代码
```
public class SocketChannelTest {

    private int size = 1024;
    private ByteBuffer byteBuffer;
    private SocketChannel socketChannel;

    public void connectServer() throws IOException {
        socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8998));
        byteBuffer = ByteBuffer.allocate(size);
        byteBuffer.order(ByteOrder.BIG_ENDIAN);
        receive();
    }

    private void receive() throws IOException {
        while (true) {
            int count;
            byteBuffer.clear();
            while ((count = socketChannel.read(byteBuffer)) > 0) {
                byteBuffer.flip();
                while (byteBuffer.hasRemaining()) {
                    System.out.print((char) byteBuffer.get());
                }
                //send("send data to server\r\n".getBytes());
                byteBuffer.clear();
            }
        }
    }

    private void send(byte[] data) throws IOException {
        byteBuffer.clear();
        byteBuffer.put(data);
        byteBuffer.flip();
        socketChannel.write(byteBuffer);
    }

    public static void main(String[] args) throws IOException {
        new SocketChannelTest().connectServer();
    }
}
```







------------------------------------

引用：[JAVA NIO][1]，[Java NIO浅析][2]

  [1]: https://www.amazon.cn/Java-NIO-Hitchens-Ron/dp/0596002882/ref=sr_1_1?ie=UTF8&qid=1493085976&sr=8-1&keywords=java+nio
  [2]:http://tech.meituan.com/nio.html
