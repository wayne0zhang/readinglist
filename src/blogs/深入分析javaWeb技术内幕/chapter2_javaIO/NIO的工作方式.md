## NIO 的工作方式
### BIO 带来的挑战
> BIO : BIO 通信模型，通常由一个独立的 Acceptor 线程负责监听客户端的连接，接受到请求之后，为每个客户端创建一个新的线程进行链路处理，处理完成之后，线程销毁。是典型的 请求-应答通信模型。

BIO 即阻塞 IO，不管是磁盘IO 还是 网络 IO，数据在写入 OutputStream 或者从 InputStream 读取时都有可能会阻塞，一旦有阻塞，线程将会失去 CPU 的使用权，这在当前的大规模访问量和有性能要求的情况下是不能被接受的。
虽然当前的网络IO 有一些解决方法，如一个客户端对应一个处理线程，出现阻塞时只是一个线程阻塞而不会影响到其他线程的工作；还有问了减少系统线程开销，使用线程池的办法减少线程创建和回收成本，但是在一些场景下仍然无法解决。
如:当前一些需要大量 HTTP 长连接的情况,像淘宝现在使用的 web 旺旺,服务端需要同时保持几百万的 HTTP 连接,但并不是每时每刻这些连接都需要传输数据,在这种情况下不可能同时创建这么多线程来保持连接.
即使线程数量不是问题,也仍然会有一些问题无法避免：比如我们想给某些客户端更高的服务优先级，很难通过设计线程的优先级来完成。
另外一种情况是，每个客户端的请求在服务端可能需要访问一些竞争资源，这些客户端在不同的线程中，因此需要同步，要实现这种同步操作远比单线程复杂得多。以上这些情况都说明，我们需要一种新的 IO 操作方式。

### NIO 的工作机制
首先看下 NIO 相关的类图：
![NIO 类图](https://ws4.sinaimg.cn/large/006tNc79gy1g1srzbhlcpj316y0tsaav.jpg)
图中有两个关键的类：Selector 和 Channel，他们是 NIO 中两个核心概念。

我们使用城市交通工具来描述 NIO 的工作方式。这里的 Channel 比 Socket 更加具体，可以比喻为某种具体的交通工具，如汽车，而把 Selector 理解为车辆运行调度系统，它负责监控每辆车的运行状态，是未出站还是在路上等。也就是 Selector 可以轮询每个 Channel 的状态。这里还有一个 Buffer 类，它比 Stream 更加具体的概念，Stream 代表座位，但没有描述座位是什么，以及是否还有座位。而 NIO 引入了 Channel、Selector 、Buffer 就是想把这些信息具体化，让程序员有机会去控制他们。

例如：当我们调用 write() 方法向 sendQ 中写入数据时，当一次写入的数据超过了 SendQ 的长度时，需要按照 SendQ 的长度进行分割，在这个过程中需要将用户地址和内核地址空间进行切换，而这个切换是你不能控制的，但在 Buffer 中我们可以自定义 Buffer 容量、是否扩容以及如何扩容等。

下面 show me the code，看下实际是怎么工作的：
```java
public void selector() throws IOException {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        Selector selector = Selector.open();
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.configureBlocking(false);//设置为非阻塞方式
        ssc.socket().bind(new InetSocketAddress(8080));
        ssc.register(selector, SelectionKey.OP_ACCEPT);// 注册监听的事件
        while(true){
            Set selectedKeys = selector.selectedKeys();// 获取所有 key 集合
            Iterator it = selectedKeys.iterator();
            while (it.hasNext()) {
                SelectionKey key = (SelectionKey) it.next();
                if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {
                    ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                    SocketChannel accept = channel.accept();
                    accept.configureBlocking(false);
                    accept.register(selector, SelectionKey.OP_READ);
                    it.remove();
                } else if ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    while (true) {
                        buffer.clear();
                        int n = channel.read(buffer);
                        if (n <= 0) {
                            break;
                        }
                        buffer.flip();
                    }
                    it.remove();
                }
            }
        }

    }
```
流程简析如下：调用 Selector 的静态方法创建一个选择器，创建一个服务端的 Channel，绑定到一个 Socket 对象，并把这个通信信道注册到选择器上，然后把通信信道设置为“非阻塞”模式。

**然后就可以调用 Selector 的 selectedKeys 方法检查已经注册在这个选择器上的所有通信信道是否有事件发生。**

如果有，将会返回所有的 SelectionKey ，通过这个对象的 channel() 方法可以获取对应的通信信道对象，从而读取通信的数据，而这里读取的数据是 buffer，这个 Buffer 就是我们可以控制的缓冲器。

在上面的这段程序中，将 Server 端的 **监听连接请求**的事件和**处理请求**的事件放在一个线程中，但是在事件应用中，我们通常会把他们放在两个线程中：一个线程专门负责监听客户端的连接请求，而且是以阻塞方式执行；另外一个线程专门负责处理请求，这个专门负责处理请求的线程才会真正采用 NIO 的方式，像 web 服务器 Tomcat 和 Jetty 都是使用这个处理方式。

Selector 可以同时监听一组通信信道（Channel）上的 IO 状态，前提是这个 Selector 已经注册到这些信道中。Selector 的 select() 方法检查已经注册的通信信道上 IO 是否已经准备好，如果没有至少一个信道 IO 状态有变化，那么 select 方法会阻塞等待或在超时时间后返回0。如果有多个信道有数据，那么将会把这些数据分配到对应的数据 Buffer 中。所以关键的地方是，有一个线程来处理所有连接的数据交互，每个连接的数据交互都不是阻塞方式，所以可同时处理大量的连接请求。

### Buffer 的工作方式
前面介绍了 Selector 检测到通信信道 IO 有数据传输时，通过 select 方法取得 SocketChannel，将数据读取或写入 Buffer 缓冲区，下面介绍 Buffer 如何接收和写出数据。

简单理解：把 Buffer 当成是一个基本数据类型的元素列表，它通过几个变量来保存列表中数据的当前位置状态，有4个值--capacity、limit、position、mark
- capacity ：缓冲区数组的长度
- position：下一个要操作的数据元素位置
- limit：缓冲区数组中不可操作的下一个元素的位置，limit<=capacity
- mark：用于记录上一次 position 的位置或者默认为0

![byteBuffer 位置图](https://ws1.sinaimg.cn/large/006tNc79gy1g1tvqy1ysfj30ua0jon1w.jpg)
1、起始位置：position = 0，capacity=limit=length 数组长度

2、写入3个数据之后，如图3；

3、图3 到 图4 是调用了 flip 方法之后，position 回到起始位置，limit取原position值，此时可以从缓冲区中正确读取这3个数据；

4、在下一次写入数据之前，调用 clear 方法，缓冲区的索引状态回到初始位置；

mark的作用呢？当我们调用 mark 方法时，它会记录当前 position 的前一个位置，当我们调用 reset 时，position 会恢复 mark 记录下来的值。

**注意**：通过 Channel 获取 IO 数据首先要经过操作系统的 Socket 缓冲区，再将数据复制到 Buffer 中，这个操作系统缓冲区就是底层 TCP 所关联的 RecvQ 或者 SendQ 队列，从操作系统缓冲区到用户缓冲区复制数据比较耗费性能，Buffer 提供了一种直接操作操作系统缓冲区的方式，即 ByteBuffer.allocateDirector(size)，这个方法返回的 DirectByteBuffer 就是底层存储空间关联的缓冲区，它通过 Native 代码操作非 JVM 堆的内存空间。每次创建或释放的时候都调用一次 System.gc() 。注意，使用 DirectByteBuffer 可能会引起内存泄漏的问题。在数据量大，生命周期较长的情况下比较合适。

### NIO 的数据访问方式
NIO 提供了两种访问文件的优化方法，FileChannel.transferTo/FileChannel.transferFrom;另一个是 FileChannel.map

- FileChannel.transferXXX
与传统方式相比，减少了数据从内核到用户空间的复制，数据直接在内核空间中移动，在linux 中使用 sendFile 系统调用。

- FileChannel.map
FileChannel.map 将文件按照一定大小块映射为内存区域，当程序访问这个内存区域时，将直接操作这个文件数据，这种方式省去了数据从内核空间向用户空间复制的损耗。**这种方式适合对大文件的只读性操作，如文件的MD5校验**。炼丹师这种方式是和操作系统的底层 IO 实现相关，如以下代码：
```java
public static void map(String[] args) {
        int BUFFER_SIZE = 1024;
        String fileName = "test.db";
        long fileLength = new File(fileName).length();
        int bufferCount = 1 + (int) (fileLength / BUFFER_SIZE);
        MappedByteBuffer[] buffers = new MappedByteBuffer[bufferCount];
        long remaining = fileLength;
        for (int i=0;i<bufferCount;i++) {
            RandomAccessFile file;
            try {
                file = new RandomAccessFile(fileName, "r");
                buffers[i] = file.getChannel().map(FileChannel.MapMode.READ_ONLY, i * BUFFER_SIZE, Math.min(remaining, BUFFER_SIZE));
            } catch (Exception e) {
                e.printStackTrace();
            }
            remaining -= BUFFER_SIZE;
        }
    }
```
