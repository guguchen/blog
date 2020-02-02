

### I/O模型基本介绍

* I/O 模型简单的理解：就是用什么样的通道进行数据的发送和接收，很大程度上决定了程序通信的性能
* Java共支持3种网络编程模型/IO模式：BIO、NIO、AIO
* Java BIO ： 同步并阻塞(**传统阻塞型**)，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销 
* Java NIO ： **同步非阻塞**，服务器实现模式为一个线程处理多个请求(连接)，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求就进行处理 
* Java AIO(NIO.2) ： **异步非阻塞**，AIO 引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用

BIO、NIO、AIO适用场景分析

1、BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序简单易理解。
2、NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯等。编程比较复杂，JDK1.4开始支持。
3、AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK7开始支持

### BIO模型基本介绍
> Java BIO 就是传统的java io 编程，其相关的类和接口在 java.io 
BIO(blocking I/O) ： 同步阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善(实现多个客户连接服务器)。 

​	BIO编程简单流程

* 服务器端启动一个ServerSocket
* 客户端启动Socket对服务器进行通信，默认情况下服务器端需要对每个客户 建立一个线程与之通讯
* 客户端发出请求后, 先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝
* 如果有响应，客户端线程会等待请求结束后，在继续执行

```java
package com.netty;

import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BIOServer {

	public static void main(String[] args) throws Exception {
		
		//创建一个线程池,每连接到一个客户端，就启动一个线程和客户端进行通信
		ExecutorService newCachedThreadPool = Executors.newCachedThreadPool();
		
		@SuppressWarnings("resource")
		ServerSocket server=new ServerSocket(6666);
		System.out.println("tomcat服务器启动...");
		while(true){
			//阻塞， 等待客户端连接
			final Socket socket = server.accept();
			System.out.println("连接到一个客户端！");
			newCachedThreadPool.execute(new Runnable() {
				
				@Override
				public void run() {
					//业务处理
					handler(socket);
				}
			});
		}
		
	}
	
	/**
	 * 处理
	 * @param socket
	 */
	public static void handler(Socket socket){
			try {
				byte[] bytes = new byte[1024];
				InputStream inputStream = socket.getInputStream();
				
				while(true){
					//读客户端数据 阻塞
					int read = inputStream.read(bytes);
					if(read != -1){
						System.out.println(new String(bytes, 0, read));
					}else{
						break;
					}
				}
				
			} catch (Exception e) {
				e.printStackTrace();
			}finally{
				try {
					System.out.println("关闭和client的连接..");
					socket.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
	}

}

```

每个请求都需要创建独立的线程，与对应的客户端进行数据 Read，业务处理，数据 Write 。

当并发数较大时，需要**创建大量线程来处理连接**，系统资源占用较大。

连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费

### NIO基本介绍

* Java NIO 全称 java non-blocking IO，是指 JDK 提供的新 API。从 JDK1.4 开始，Java 提供了一系列改进的输入/输出的新特性，被统称为 NIO(即 New IO)，是**同步非阻塞**的

* NIO 相关类都被放在 java.nio 包及子包下，并且对原 java.io 包中的很多类进行改写。【基本案例】

* NIO 有三大核心部分：**Channel(通道)**，**Buffer(缓冲区)**, **Selector(选择器)** 

* NIO是 面向**缓**冲区 ，或者面向**块** 编程的。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供**非阻塞**式的高伸缩性网络

* Java NIO的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情

  | IO                      | NIO                        |
  | ----------------------- | -------------------------- |
  | 面向流(Stream Oriented) | 面向缓冲区(Buffer Oriented |
  | 阻塞IO(Blocking IO）    | 非阻塞IO(Non Blocking IO)  |
  | (无)                    | 选择器(Selectors)          |

<img src="C:\Users\chenzhihao\AppData\Roaming\Typora\typora-user-images\image-20200202151539623.png" alt="image-20200202151539623"  />

1、每个channel 都会对应一个Buffer

2、Selector 对应一个线程， 一个线程对应多个channel(连接)

3、该图反应了有三个channel 注册到 该selector //程序

4、程序切换到哪个channel 是有事件决定的, Event 就是一个重要的概念

5、Selector 会根据不同的事件，在各个通道上切换

6、Buffer 就是一个内存块 ， 底层是有一个数组

7、数据的读取写入是通过Buffer, 这个和BIO , BIO 中要么是输入流，或者是
 输出流, 不能双向，但是NIO的Buffer 是可以读也可以写, 需要 flip 方法切换

8、channel 是双向的, 可以返回底层操作系统的情况, 比如Linux ， 底层的操作系统
 通道就是双向的.

### 缓冲区

> 缓冲区（Buffer）：缓冲区本质上是一个可以读写数据的内存块，可以理解成是一个**容器对象**(含数组)**，该对象提供了**一组方法，可以更轻松地使用内存块，，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。Channel 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由 Buffer，

在 NIO 中，Buffer 是一个顶层父类，它是一个抽象类

```java
ByteBuffer,存储字节数据到缓冲区
ShortBuffer，存储字符串数据到缓冲区
CharBuffer，存储字符数据到缓冲区
IntBuffer，存储整数数据到缓冲区
LongBuffer，存储长整型数据到缓冲区
DoubleBuffer，存储小数到缓冲区
FloatBuffer，存储小数到缓冲区
```

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 容量，即可以容纳的最大数据量；在缓冲区创建时被设定并且不能改变 |
| Limit    | 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变改值，为下次读写作准备 |
| Mark     | 标记                                                         |

API

```java
public abstract class Buffer {
    //JDK1.4时，引入的api
    public final int capacity( )//返回此缓冲区的容量
    public final int position( )//返回此缓冲区的位置
    public final Buffer position (int newPositio)//设置此缓冲区的位置
    public final int limit( )//返回此缓冲区的限制
    public final Buffer limit (int newLimit)//设置此缓冲区的限制
    public final Buffer mark( )//在此缓冲区的位置设置标记
    public final Buffer reset( )//将此缓冲区的位置重置为以前标记的位置
    public final Buffer clear( )//清除此缓冲区, 即将各个标记恢复到初始状态，但是数据并没有真正擦除, 后面操作会覆盖
    public final Buffer flip( )//反转此缓冲区
    public final Buffer rewind( )//重绕此缓冲区
    public final int remaining( )//返回当前位置与限制之间的元素数
    public final boolean hasRemaining( )//告知在当前位置和限制之间是否有元素
    public abstract boolean isReadOnly( );//告知此缓冲区是否为只读缓冲区
 
    //JDK1.6时引入的api
    public abstract boolean hasArray();//告知此缓冲区是否具有可访问的底层实现数组
    public abstract Object array();//返回此缓冲区的底层实现数组
    public abstract int arrayOffset();//返回此缓冲区的底层实现数组中第一个缓冲区元素的偏移量
    public abstract boolean isDirect();//告知此缓冲区是否为直接缓冲区
}

```
最常用的是ByteBuffer
```java
public abstract class ByteBuffer {
    //缓冲区创建相关api
    public static ByteBuffer allocateDirect(int capacity)//创建直接缓冲区
    public static ByteBuffer allocate(int capacity)//设置缓冲区的初始容量
    public static ByteBuffer wrap(byte[] array)//把一个数组放到缓冲区中使用
    //构造初始化位置offset和上界length的缓冲区
    public static ByteBuffer wrap(byte[] array,int offset, int length)
     //缓存区存取相关API
    public abstract byte get( );//从当前位置position上get，get之后，position会自动+1
    public abstract byte get (int index);//从绝对位置get
    public abstract ByteBuffer put (byte b);//从当前位置上添加，put之后，position会自动+1
    public abstract ByteBuffer put (int index, byte b);//从绝对位置上put
 }
```

#### 直接缓冲区与非直接缓冲区

直接与非直接缓冲区
*  字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在 此缓冲区上执行本机 I/O 操作。也就是说，在每次调用基础操作系统的一个本机 I/O 操作之前（或之后）， 虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。
*  直接字节缓冲区可以通过调用此类的 allocateDirect() 工厂方法来创建。此方法返回的缓冲区进行分配和取消 分配所需成本通常高于非直接缓冲区。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对 应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的 本机 I/O 操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好 处时分配它们。
*   直接字节缓冲区还可以通过FileChannel 的 map() 方法 将文件区域直接映射到内存中来创建。该方法返回 MappedByteBuffer 。Java 平台的实现有助于通过 JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区 中的某个缓冲区实例指的是不可访问的内存区域，则试图访问该区域不会更改该缓冲区的内容，并且将会在 访问期间或稍后的某个时间导致抛出不确定的异常。
*  字节缓冲区是直接缓冲区还是非直接缓冲区可通过调用其 isDirect() 方法来确定。提供此方法是为了能够在 性能关键型代码中执行显式缓冲区管理。

### 通道基本介绍

> 1、NIO的通道类似于流，但有些区别如下：
> 	通道可以同时进行读写，而流只能读或者只能写
> 	通道可以实现异步读写数据
> 	通道可以从缓冲读数据，也可以写数据到缓冲: 
> 2、BIO 中的 stream 是单向的，例如 FileInputStream 对象只能进行读取数据的操作，而 NIO 中的通道(Channel)是双向的，可以读操作，也可以写操作。
> 3、Channel在NIO中是一个接口```public interface Channel extends Closeable{} ```不用在finally块显式关闭
> 4、 常用的 Channel 类有：FileChannel、DatagramChannel、ServerSocketChannel 和 SocketChannel。【ServerSocketChanne 类似 ServerSocket , SocketChannel 类似 Socket】

![image-20200202153807631](C:\Users\chenzhihao\AppData\Roaming\Typora\typora-user-images\image-20200202153807631.png)

将1.txt中的内容用通道写到2.txt

```java
package com.netty.nio;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class FileChannel3 {
    public static void main(String[] args) throws IOException {
        FileInputStream fileInputStream = new FileInputStream("1.txt");
        FileChannel fileInputStreamChannel = fileInputStream.getChannel();
        FileOutputStream fileOutputStream = new FileOutputStream("2.txt");
        FileChannel fileOutputStreamChannel = fileOutputStream.getChannel();
        ByteBuffer byteBuffer=ByteBuffer.allocate(512);
        while(true){
            byteBuffer.clear();
            int read = fileInputStreamChannel.read(byteBuffer);
            if(read==-1)
                break;
            byteBuffer.flip();
            fileOutputStreamChannel.write(byteBuffer);

        }
        fileInputStream.close();
        fileOutputStream.close();    }
}


/**
public int read(ByteBuffer dst) ，从通道读取数据并放到缓冲区中
public int write(ByteBuffer src) ，把缓冲区的数据写到通道中
public long transferFrom(ReadableByteChannel src, long position, long count)，从目标通道中复制数据到当前通道
public long transferTo(long position, long count, WritableByteChannel target)，把数据从当前通道复制给目标通道

*/
```

```java
package com.netty;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.channels.FileChannel;

public class NIOFileCopy {

	public static void main(String[] args) throws Exception {
		
		FileInputStream fis=new FileInputStream("d:\\Koala.jpg");
		FileOutputStream fos=new FileOutputStream("d:\\Koala2.jpg");
		/*
		 * 说明
		 * 从两个流中得到两个通道，sourCh ,destCh ，
		 * 然后直接调用 transferFrom 完成文件复制
		 */
		FileChannel sourCh = fis.getChannel();
		FileChannel destCh = fos.getChannel();
		
		/*
		 * transferFrom 方法可以将两个通道连接起来，进行数据传输
	 * @param  src
     *         The source channel
     *
     * @param  position
     *         The position within the file at which the transfer is to begin;
     *         must be non-negative
     *
     * @param  count
     *         The maximum number of bytes to be transferred; must be
     *         non-negative
     *
     * @return  The number of bytes, possibly zero,
     *          that were actually transferred
		 */
		
		destCh.transferFrom(sourCh, 0, sourCh.size()); 
		sourCh.close();
		destCh.close();
		fis.close();
		fos.close();
		System.out.println("图片拷贝完毕~~");
	}

}

```

注意：

ByteBuffer 支持类型化的put 和 get, put 放入的是什么数据类型，get就应该使用相应的数据类型来取出，否则可能有 BufferUnderflowException 异常。

可以将一个普通Buffer 转成只读Buffer

NIO 还提供了 MappedByteBuffer， 可以让文件直接在内存（堆外的内存）中进行修改， 而如何同步到文件由NIO 来完成

```java
ackage com.netty.nio;

import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

/**
 * 可让文件直接在堆外内存修改，操作系统不需要拷贝一份
 */
public class MappedByteBufferTest {
    public static void main(String[] args) throws IOException {
        RandomAccessFile file = new RandomAccessFile("1.txt", "rw");
        FileChannel channel = file.getChannel();
        //模式，起始位置，大小（字节）
        MappedByteBuffer map = channel.map(FileChannel.MapMode.READ_WRITE, 0, 5);
        map.put(0, (byte) 'H');
        map.put(3, (byte) '9');
        file.close();

    }
}
```



前面我们讲的读写操作，都是通过一个Buffer 完成的，NIO 还支持 通过多个Buffer (即 Buffer 数组) 完成读写操作，即 Scattering 和 Gathering 

```java
ByteBuffer[] byteBuffers = new ByteBuffer[2];
        byteBuffers[0]=ByteBuffer.allocate(5);
        byteBuffers[1]= ByteBuffer.allocate(3);

        SocketChannel socketChannel = serverSocketChannel.accept();
        while (true){
            int read=0;
            while(read<8){
                long read1 = socketChannel.read(byteBuffers);
                read+=read1;
                System.out.println(read1);
                Arrays.asList(byteBuffers).stream().map(buffer -> buffer.position()+","+buffer.limit()).forEach(
                        System.out::println
                );
            }
            Arrays.asList(byteBuffers).stream().forEach(buffer->buffer.flip());
            long write=0;
            while(write<8){
                long l = socketChannel.write(byteBuffers);
                write+=l;
            }
            Arrays.asList(byteBuffers).stream().forEach(buffer->buffer.clear());


        }
```

### Selector基本介绍

> 1)Java 的 NIO，用非阻塞的 IO 方式。可以用一个线程，处理多个的客户端连接，就会使用到**Selector(选择器)**
>
> 2)**Selector** **能**够检测多个注册的通道上是否有事件发生(注意:多个Channel以事件的方式可以注册到同一个Selector)，如果有事件发生，便获取事件然后针对每个事件进行相应的处理。这样就可以只用一个单线程去管理多个通道，也就是管理多个连接和请求。
>
> 3)只有在 连接/通道 真正有读写事件发生时，才会进行读写，就大大地减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程
>
> 4)避免了多线程之间的上下文切换导致的开销

```java
public abstract class Selector implements Closeable { 
public static Selector open();//得到一个选择器对象
public int select(long timeout);//监控所有注册的通道，当其中有 IO 操作可以进行时，将
对应的 SelectionKey 加入到内部集合中并返回，参数用来设置超时时间
public Set<SelectionKey> selectedKeys();//从内部集合中得到所有的 SelectionKey	
}

```

selector 是 SelectableChannle 对象的多路复用器，Selector 可 以同时监控多个 SelectableChannel 的 IO 状况

![image-20200202160746674](C:\Users\chenzhihao\AppData\Roaming\Typora\typora-user-images\image-20200202160746674.png)

1) SelectionKey，表示 Selector 和网络通道的注册关系, 共四种:

int OP_ACCEPT：有新的网络连接可以 accept，值为 16

int OP_CONNECT：代表连接已经建立，值为 8

int OP_READ：代表读操作，值为 1 

int OP_WRITE：代表写操作，值为 4

```java
public abstract class SelectionKey {
     public abstract Selector selector();//得到与之关联的 Selector 对象
 public abstract SelectableChannel channel();//得到与之关联的通道
public final Object attachment();//得到与之关联的共享数据
public abstract SelectionKey interestOps(int ops);//设置或改变监听事件
 public final boolean isAcceptable();//是否可以 accept
public final boolean isReadable();//是否可以读
 public final boolean isWritable();//是否可以写
}

```
**NIO服务器**

```java
package com.netty.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void main(String[] args) throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        Selector selector = Selector.open();

        serverSocketChannel.socket().bind(new InetSocketAddress(6666));

        serverSocketChannel.configureBlocking(false);
        //注意同通道一定要打开非阻塞模式

        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while(true){
            if(selector.select(1000)==0){
                System.out.println("服务器等待了一秒");
                continue;
            }
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while(iterator.hasNext()) {
                SelectionKey key = iterator.next();

                if (key.isAcceptable()) {
                    SocketChannel accept = serverSocketChannel.accept();
                    accept.configureBlocking(false);
                    accept.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));

                }
                if (key.isReadable()) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    channel.configureBlocking(false);
                    ByteBuffer attachment = (ByteBuffer) key.attachment();
                    channel.read(attachment);
                    System.out.println("from client" + new String(attachment.array()));

                }
                iterator.remove();
            }


        }

    }
}


package com.netty.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectableChannel;
import java.nio.channels.SocketChannel;

public class NIOClient {
    public static void main(String[] args) throws IOException {
        SocketChannel open = SocketChannel.open();

         open.configureBlocking(false);

         if(!open.connect(new InetSocketAddress("127.0.0.1",6666))){
             while(!open.finishConnect()){
                 System.out.println("因为连接需要时间，客户端不会阻塞，可以做其它工作");
             }
         }
         String str="hello world";

        ByteBuffer byteBuffer = ByteBuffer.wrap(str.getBytes());

        open.write(byteBuffer);
        System.in.read();

    }
}

```

**NIO群聊系统**

```java
package com.netty.nio.groupchat;


import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.text.SimpleDateFormat;
import java.util.Iterator;


public class Server {
    private Selector selector;
    private ServerSocketChannel listenerChannel;
    private static final int PORT = 6677; //服务器端口

    public Server() {
        try {
            // 得到选择器
            selector = Selector.open();
            // 打开监听通道
            listenerChannel = ServerSocketChannel.open();
            // 绑定端口
            listenerChannel.socket().bind(new InetSocketAddress(PORT));
            // 设置为非阻塞模式
            listenerChannel.configureBlocking(false);
            // 将选择器绑定到监听通道并监听 accept 事件
            listenerChannel.register(selector, SelectionKey.OP_ACCEPT);
            printInfo("服务器 ok.......");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void listen() {
        try {
            while (true) { //不停轮询
                int count = selector.select();//获取就绪 channel
                if (count > 0) {
                    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                    while (iterator.hasNext()) {
                        SelectionKey key = iterator.next();
                        // 监听到 accept
                        if (key.isAcceptable()) {
                            SocketChannel sc = listenerChannel.accept();
                            //非阻塞模式
                            sc.configureBlocking(false);
                            //注册到选择器上并监听 read
                            sc.register(selector, SelectionKey.OP_READ);

                            //System.out.println(sc.getRemoteAddress().toString().substring(1) + "online ...");
                            System.out.println(sc.socket().getRemoteSocketAddress().toString().substring(1) + " 上线 ...");
                            //将此对应的 channel 设置为 accept,接着准备接受其他客户端请求
                            key.interestOps(SelectionKey.OP_ACCEPT);
                        }
                        //监听到 read
                        if (key.isReadable()) {
                            readData(key); //读取客户端发来的数据
                        }
                        //一定要把当前 key 删掉，防止重复处理
                        iterator.remove();
                    }
                } else {
                    System.out.println("waitting ...");
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void readData(SelectionKey key) {
        SocketChannel channel = null;
        try {
            // 得到关联的通道
            channel = (SocketChannel) key.channel();
            //设置 buffer 缓冲区
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            //从通道中读取数据并存储到缓冲区中
            int count = channel.read(buffer);
            //如果读取到了数据
            if (count > 0) {
                //把缓冲区数据转换为字符串
                String msg = new String(buffer.array());

                printInfo(msg);
                //将关联的 channel 设置为 read，继续准备接受数据
                key.interestOps(SelectionKey.OP_READ);
                sendInfoToOtherClients(channel, msg); //向所有客户端广播数据
            }
            buffer.clear();
        } catch (IOException e) {
            try {
                //当客户端关闭 channel 时，进行异常如理
                //printInfo(channel.getRemoteAddress().toString().substring(1) + "offline...");
                printInfo(channel.socket().getRemoteSocketAddress().toString().substring(1) + " 离线了 ...");
                key.cancel(); //取消注册
                channel.close(); //关闭通道
            } catch (IOException e1) {
                e1.printStackTrace();
            }
        }
    }

    public void sendInfoToOtherClients(SocketChannel except, String msg) throws IOException {
        System.out.println("服务器进行消息转发 ...");
        //转发数据到所有的 SocketChannel 中
        for (SelectionKey key : selector.keys()) {
            Channel targetchannel = key.channel();
            //排除自身
            if (targetchannel instanceof SocketChannel && targetchannel != except) {
                SocketChannel dest = (SocketChannel) targetchannel;
                //把数据存储到缓冲区中
                ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
                //往通道中写数据
                dest.write(buffer);
            }
        }
    }

    private void printInfo(String str) { //显示消息

        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        System.out.println("服务器接收到消息 时间: [" + sdf.format(new java.util.Date()) + "] -> " + str);
    }

    public static void main(String[] args) {
        Server server = new Server();
        server.listen();
    }
}



package com.netty.nio.groupchat;



import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Scanner;
import java.util.Set;

public class Client {
    private final String HOST = "127.0.0.1"; //服务器地址
    private int PORT = 6677; //服务器端口
    private Selector selector;
    private SocketChannel socketChannel;
    private String userName;

    public Client() throws IOException {
        //得到选择器
        selector = Selector.open();
        //连接远程服务器
        socketChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", PORT));
        //设置非阻塞
        socketChannel.configureBlocking(false);
        //注册选择器并设置为 read
        socketChannel.register(selector, SelectionKey.OP_READ);
        //得到客户端 IP 地址和端口信息，作为聊天用户名使用
        userName = socketChannel.getLocalAddress().toString().substring(1);
        System.out.println(userName + " is ok ~");
    }

    //向服务器端发送数据
    public void sendInfo(String info) throws Exception {
        //如果控制台输入 exit 就关闭通道，结束聊天
        if (info.equalsIgnoreCase("exit")) {
            socketChannel.write(ByteBuffer.wrap(info.getBytes()));
            socketChannel.close();
            socketChannel = null;
            return;
        }
        info = userName + " 说: " + info;
        try {
            //往通道中写数据
            socketChannel.write(ByteBuffer.wrap(info.getBytes()));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //从服务器端接收数据
    public void readInfo() {
        try {
            int readyChannels = selector.select();
            if (readyChannels > 0) { //有可用通道
                Set selectedKeys = selector.selectedKeys();
                Iterator keyIterator = selectedKeys.iterator();
                while (keyIterator.hasNext()) {
                    SelectionKey sk = (SelectionKey) keyIterator.next();
                    if (sk.isReadable()) {
                        //得到关联的通道
                        SocketChannel sc = (SocketChannel) sk.channel();
                        //得到一个缓冲区
                        ByteBuffer buff = ByteBuffer.allocate(1024);
                        //读取数据并存储到缓冲区
                        sc.read(buff);
                        //把缓冲区数据转换成字符串
                        String msg = new String(buff.array());
                        System.out.println(msg.trim());
                    }
                    keyIterator.remove(); //删除当前 SelectionKey，防止重复处理
                }
            } else {
                //会检测到没有可用的channel ，可以退出
                System.out.println("没有可用channel ...");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception  {
        //创建一个聊天客户端对象
        Client chatClient = new Client();
        new Thread() { //单独开一个线程不断的接收服务器端广播的数据
            public void run() {
                while (true) {
                    chatClient.readInfo();
                    try { //间隔 3 秒
                        Thread.currentThread().sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();

        Scanner scanner = new Scanner(System.in);
        //在控制台输入数据并发送到服务器端
        while (scanner.hasNextLine()) {
            String msg = scanner.nextLine();
            chatClient.sendInfo(msg.trim());
        }
    }
}

```

### 零拷贝

​		*传统IO*

![image-20200202162431004](C:\Users\chenzhihao\AppData\Roaming\Typora\typora-user-images\image-20200202162431004.png)

​		**mmap** **优化**

mmap 通过内存映射，将文件映射到内核缓冲区，同时，用户空间可以共享内核空间的数据。这样，在进行网络传输时，就可以减少内核空间到用户控件的拷贝

![image-20200202162540350](C:\Users\chenzhihao\AppData\Roaming\Typora\typora-user-images\image-20200202162540350.png)

sendFile优化

数据根本不经过用户态，直接从内核缓冲区进入到 Socket Buffer，同时，由于和用户态完全无关，就减少了一次上下文切换。

![image-20200202162754912](C:\Users\chenzhihao\AppData\Roaming\Typora\typora-user-images\image-20200202162754912.png)

**mmap 适合小数据量读写，sendFile 适合大文件传输**，零拷贝不是没有拷贝，NIO 零拷贝方式传递(transferTo)

### AIO

[AIO介绍](http://www.52im.net/thread-306-1-1.html)