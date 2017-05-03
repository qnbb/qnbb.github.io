---
layout: post
title: Socket 通道
categories: [Java, NIO]
description: Socket 通道
keywords: JAVA, NIO
---


Channel 用于在字节缓冲区和位于通道另一侧的实体(通常是一个文件或套接字)之间有效地传输数据。通道可以形象地比喻为银行出纳窗口使用的气动导管。您的薪水支票就是您要传送的信息,载体(Carrier)就好比一个缓冲区。您先填充缓冲区(将您的支票放到载体上),接着将缓冲“写”到通道中(将载体丢进导管中),然后信息负载就被传递到通道另一侧的 I/O 服务(银行出纳员)。

该过程的回应是:出纳员填充缓冲区(将您的收据放到载体上),接着开始一个反方向的通道传输(将载体丢回到导管中)。载体就到了通道的您这一侧(一个填满了的缓冲区正等待您的查验),然后您就会 flip 缓冲区(打开盖子)并将它清空(移除您的收据)。现在您可以开车走了,下一个对象(银行客户)将使用同样的载体(Buffer)和导管(Channel)对象来重复上述过程。

DatagramChannel 和 SocketChannel 实现定义读和写功能的接口而ServerSocketChannel不实现。ServerSocketChannel 负责监听传入的连接和创建新的 SocketChannel 对象,它本身从不传输数据。

在我们具体讨论每一种 socket 通道前,您应该了解 socket 和 socket 通道之间的关系。之前的章节中有写道,通道是一个连接 I/O 服务导管并提供与该服务交互的方法。就某个 socket 而言,它不会再次实现与之对应的 socket 通道类中的 socket 协议 API,而 java.net 中已经存在的 socket 通道都可以被大多数协议操作重复使用。

全部 socket 通道类(DatagramChannel、SocketChannel 和ServerSocketChannel)在被实例化时都会创建一个对等 socket 对象。这些是我们所熟悉的来自 java.net 的类(Socket、ServerSocket和 DatagramSocket),它们已经被更新以识别通道。对等 socket 可以通过调用 socket( )方法从一个通道上获取。此外,这三个 java.net 类现在都有 getChannel( )方法。

#### ServerSocketChannel

```
public abstract class ServerSocketChannel
extends AbstractSelectableChannel
{
public static ServerSocketChannel open( ) throws IOException
public abstract ServerSocket socket( );
public abstract ServerSocket accept( ) throws IOException;
public final int validOps( )
}
```
ServerSocketChannel 是一个基于通道的 socket 监听器。它同我们所熟悉的 java.net.ServerSocket执行相同的基本任务,不过它增加了通道语义,因此能够在非阻塞模式下运行。用静态的 open( )工厂方法创建一个新的ServerSocketChannel 对象,将会返回同一个未绑定的java.net.ServerSocket 关联的通道。该对等 ServerSocket 可以通过在返回的 ServerSocketChannel 上调用 socket( )方法来获取。

作为 ServerSocketChannel 的对等体被创建的 ServerSocket 对象依赖通道实
现。这些 socket 关联的 SocketImpl 能识别通道。通道不能被封装在随意的 socket 对象外面。

由于 ServerSocketChannel 没有 bind( )方法,因此有必要取出对等的 socket 并使用它来绑定到一个端口以开始监听连接。我们也是使用对等 ServerSocket 的 API 来根据需要设置其他的 socket 选项。

```
ServerSocketChannel ssc = ServerSocketChannel.open( );
ServerSocket serverSocket = ssc.socket( );
// Listen on port 1234
serverSocket.bind (new InetSocketAddress (1234));
```

同它的对等体 java.net.ServerSocket 一样,ServerSocketChannel 也有 accept( )方法。一旦您创建了一个 ServerSocketChannel 并用对等 socket 绑定了它,然后您就可以在其中一个上调用 accept( )。如果您选择在 ServerSocket 上调用 accept( )方法,那么它会同任何其他的 ServerSocket 表现一样的行为:总是阻塞并返回一个 java.net.Socket 对象。

**如果您选择在 ServerSocketChannel 上调用 accept( )方法则会返回SocketChannel 类型的对象,返回的对象能够在非阻塞模式下运行。**假设系统已经有一个安全管理器(security manager),两种形式的方法调用都执行相同的安全检查。

如果以非阻塞模式被调用,当没有传入连接在等时，ServerSocketChannel.accept( )会立即返回 null。正是这种检查连接而不阻塞的能力实现了可伸缩性并降低了复杂性。可选择性也因此得到实现。我们可以使用一个选择器实例来注册一个 ServerSocketChannel 对象以实现新连接到达时自动通知的功能。下面的例子演示了如何使用一个非阻塞的 accept( )方法:

```
/**
 * Created by mh on 17-4-21.
 */
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.net.InetSocketAddress;
/**
 * Test nonblocking accept( ) using ServerSocketChannel.
 * Start this program, then "telnet localhost 1234" to
 * connect to it.
 *
 */
public class ChannelAccept
{
	public static final String GREETING = "Hello I must be going.\r\n";
	public static void main (String [] argv)
			throws Exception
	{
		int port = 1234; // default
		if (argv.length > 0) {
			port = Integer.parseInt (argv [0]);
		}
		ByteBuffer buffer = ByteBuffer.wrap (GREETING.getBytes( ));
		ServerSocketChannel ssc = ServerSocketChannel.open( );
		ssc.socket( ).bind (new InetSocketAddress (port));
		ssc.configureBlocking (false);
		while (true) {
			System.out.println ("Waiting for connections " + port++);
			// 区别 ssc.socket.accept(),非阻塞，返回一个SocketChannel对象
			SocketChannel sc = ssc.accept( );
			if (sc == null) {
// no connections, snooze a while
				Thread.sleep (2000);
			} else {
				System.out.println ("Incoming connection from: "
						+ sc.socket().getRemoteSocketAddress( ));
				buffer.rewind( );
				sc.write (buffer);
				sc.close( );
			}
		}
	}
}
```
结果如下，ServerSocketChannel一直等待客户端Socket的连接：
```
Waiting for connections 1234
Waiting for connections 1235
Waiting for connections 1236
Waiting for connections 1237
Waiting for connections 1238
Waiting for connections 1239
Waiting for connections 1240
Waiting for connections 1241
...
```

#### SocketChannel

```
public abstract class SocketChannel
extends AbstractSelectableChannel
implements ByteChannel, ScatteringByteChannel, GatheringByteChannel
{
// This is a partial API listing
public static SocketChannel open( ) throws IOException
public static SocketChannel open (InetSocketAddress remote)
throws IOException
public abstract Socket socket( );
public abstract boolean connect (SocketAddress remote)
throws IOException;
public abstract boolean isConnectionPending( );
public abstract boolean finishConnect( ) throws IOException;
public abstract boolean isConnected( );
public final int validOps( )
}
```

Socket 和 SocketChannel 类封装点对点、有序的网络连接,类似于我们所熟知并喜爱的 TCP/IP网络连接。SocketChannel 扮演客户端发起同一个监听服务器的连接。直到连接成功,它才能收到数据并且只会从连接到的地址接收。

每个 SocketChannel 对象创建时都是同一个对等的 java.net.Socket 对象串联的。静态的 open( )方法可以创建一个新的 SocketChannel 对象,而在新创建的 SocketChannel 上调用 socket( )方法能返回它对等的 Socket 对象;在该 Socket 上调用 getChannel( )方法则能返回最初的那个 SocketChannel。

我们可以通过在通道上直接调用 connect( )方法或在通道关联的 Socket 对象上调用 connect( )来将该 socket 通道连接。一旦一个 socket 通道被连接,它将保持连接状态直到被关闭。您可以通过调用布尔型的 isConnected( )方法来测试某个SocketChannel 当前是否已连接。

第二种带 InetSocketAddress 参数形式的 open( )是在返回之前进行连接的便捷方法。这段代码:
```
SocketChannel socketChannel =
SocketChannel.open (new InetSocketAddress ("somehost", somePort));
```
等价于下面这段代码:
```
SocketChannel socketChannel = SocketChannel.open( );
socketChannel.connect (new InetSocketAddress ("somehost", somePort));
```

如果您选择使用传统方式进行连接——通过在对等 Socket 对象上调用 connect( )方法，那么传统的连接语义将适用于此。线程在连接建立好或超时过期之前都将保持阻塞。如果您选择通过在通道上直接调用 connect( )方法来建立连接并且通道处于阻塞模式(默认模式),那么连接过程实际上是一样的。

在 SocketChannel 上并没有一种 connect( )方法可以让您指定超时(timeout)值,当 connect( )方法在非阻塞模式下被调用时 SocketChannel 提供并发连接：它发起对请求地址的连接并且立即返回值。如果返回值是 true，说明连接立即建立了(这可能是本地环回连接)；如果连接不能立即建立,connect( )方法会返回 false 且并发地继续连接建立过程。

面向流的的 socket 建立连接状态需要一定的时间，因为两个待连接系统之间必须进行包对话，以建立维护流 socket 所需的状态信息。跨越开放互联网连接到远程系统会特别耗时。假如某个SocketChannel 上当前正由一个并发接，isConnectPending( )方法就会返回 true 值。

以下是一段用来管理异步连接的可用代码。

```
/**
 * Created by mh on 17-4-21.
 */
import java.nio.channels.SocketChannel;
import java.net.InetSocketAddress;
/**
 * Demonstrate asynchronous connection of a SocketChannel.
 * @author Ron Hitchens (ron@ronsoft.com)
 */
public class ConnectAsync
{
	public static void main (String [] argv) throws Exception
	{
		String host = "localhost";
		int port = 1234;
		if (argv.length == 2) {
			host = argv [0];
			port = Integer.parseInt (argv [1]);
		}
		InetSocketAddress addr = new InetSocketAddress (host, port);
		SocketChannel sc = SocketChannel.open( );
		sc.configureBlocking (false);
		System.out.println ("initiating connection");
		sc.connect (addr);
		while ( ! sc.finishConnect( )) {
			doSomethingUseful( );
		}
		System.out.println ("connection established");
// Do something with the connected socket
// The SocketChannel is still nonblocking
		sc.close( );
	}
	private static void doSomethingUseful( )
	{
		System.out.println ("doing something useless");
	}
}
```
和之前 ServerSocketChannel 的代码可以完成Socket连接，结果如下：

ServerSocketChannel:
```
Waiting for connections1234
Waiting for connections1235
Waiting for connections1236
Waiting for connections1237
Incoming connection from: /127.0.0.1:38252
Waiting for connections1238
Waiting for connections1239
Waiting for connections1240
...
```
SocketChannel:
```
initiating connection
connection established

Process finished with exit code 0
```

