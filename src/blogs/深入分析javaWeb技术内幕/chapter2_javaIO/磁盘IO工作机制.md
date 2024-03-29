## 磁盘IO工作机制
> ref: 《深入分析java web 技术内幕》 by：许令波
### 几种访问文件的方式

文件读取和写入的 IO 操作都是调用操作系统提供的接口，因为磁盘设备是由操作系统管理的，应用程序要访问物理设备，只能通过系统调用的方式来工作。读和写分别对应 read()/write() 两个系统调用。而只要是系统调用就可能存在内核空间地址到用户空间地址的切换的问题。这是操作系统为了保护系统本身的运行安全，而将内核程序运行使用的内存空间和用户程序使用的内存空间进行隔离造成的。但是这样虽然保证了内核程序的安全，但是也必然存在着数据可能从内核空间向用户空间赋值的问题。

如果遇到非常耗时的操作，如磁盘IO，数据从磁盘复制到内核空间，再从内核空间复制到用户空间，这个过程将会非常缓慢。这是操作系统为了加速 IO 操作，在内核空间使用了缓存机制。也就是将从磁盘读取的文件按照一定的组织方式进行缓存，如果用户程序访问的是同一段磁盘地址的空间数据，那么操作系统将从内核缓存中直接取出返回给用户程序，这样可以减小 IO 的响应时间。

- 标准访问文件的方式

标准访问文件的方式就是当应用程序调用 read 接口时，操作系统检查在内核的高速缓存中有没有需要的数据，如果已经缓存了，那么就直接从缓存中返回，如果没有，就从磁盘中获取，然后缓存在操作系统的缓存中。

写入的方式是，应用程序调用 write 接口将数据从用户地址空间赋值到内核地址空间的缓存中。这时对用户程序来说写操作就已经完成，至于操作系统什么时候写到磁盘中由操作系统决定，除非显式的调用了 sync 同步命令。
![标准访问文件的方式](https://ws2.sinaimg.cn/large/006tKfTcly1g1o4l5f9t2j30p80h6dg5.jpg)

- 直接 IO 的方式

所谓的直接 IO 的方式就是应用程序直接操作磁盘数据，而不经过操作系统内核数据缓冲区，这样做的目的是为了减小一次从内核缓冲区到用户缓存的数据复制。
**这种访问文件的方式通常是在对数据的缓存管理由应用程序实现的数据库管理系统中。**
如在数据库管理系统中，系统明确的知道应该缓存哪些数据，应该失效哪些数据，还可以对热点数据进行预加载，提前将数据加载到内存中，可以加速数据的访问效率。
如果这时由操作系统来进行缓存，则很难做到，因为操作不知道哪些是热点数据，哪些数据只会访问一次。操作系统知识简单的缓存最近一次从磁盘读取的数据。
但是直接操作 IO 也有负面影响，如果访问的数据不在应用程序缓存中，那么每次数据都会直接从磁盘加载，这种直接加载会非常的缓慢。通常直接 IO 与 异步 IO 结合使用，会得到比较好的性能。

- 同步访问文件的方式

同步访问文件的方式比较好理解，就是数据的读取和写入都是同步操作，与标准访问方式不同是：只有数据真正被写入到磁盘中之后，才会返回给应用程序成功的标志。
**这种访问文件的方式性能比较差，只有在一些对数据安全性要求较高的场景中才会使用，而且这种操作方式的硬件都是定制的。**

- 异步访问文件的方式

异步访问文件的方式就是当访问数据的线程发出请求之后，线程会接着去处理其他事情，而不是阻塞等待，当请求的数据返回后继续处理下面的操作。
**这种访问文件的方式可以明显的提高应用程序的效率，但是不会改变访问文件的效率。**
![异步方式](https://ws4.sinaimg.cn/large/006tKfTcly1g1o4mq9ae3j30pw0gqaad.jpg)

- 内存映射的方式

内存映射的方式是指操作系统将内存中的某一块区域和磁盘中的文件关联起来，当要访问内存中的一段数据时，转换为访问文件的某一段数据。这种方式的目的同样是减少数据从内核空间缓存到用户空间缓存的数据复制操作，因为这两个空间是共享的。
![内存映射方式](https://ws1.sinaimg.cn/large/006tKfTcly1g1o4ou9bl5j30sg0gwdg8.jpg)

### java 访问磁盘文件

下面介绍一下将数据持久化到物理磁盘的方式。
我们知道数据在磁盘中的最小描述就是文件，也就是说上层应用程序只能通过文件来操作磁盘上的数据，文件也是操作系统和磁盘驱动器交互的最小单元。值得注意的是，在 java 中通常的 File 并不代表一个真实存在的文件对象，当你指定一个路径描述符时，他就会返回代表这个路径的虚拟对象，它可能是一个真实存在的文件或者一个包含多个文件的文件夹。为什么要这样设计呢？因为多数情况下，我们并不关心这是文件是否存在，我们只关心这个文件到底如何操作。那么什么时候会检查这个文件是否真实存在呢？
**只有在真正需要读取文件的时候，才会去检查这个文件是否存在**
例如：FileInputStream 类是操作一个文件的接口，注意到在创建一个 FileInputStream 对象时会创建一个 FileDescritor 对象，其实这个对象就是真正代表一个存在的文件对象的描述。当我们在操作一个文件对象 时，可以通过 getFD() 方法来获取真正操作的与底层操作系统相关联的文件描述。例如，可以通过 FileDescriptor.sync() 方法将操作系统缓存中的数据强制刷新到物理磁盘中。

![文件读取示例](https://ws1.sinaimg.cn/large/006tKfTcly1g1o4wx3azoj319g0iyq3k.jpg)
当传入一个文件路径时，将会根据这个路径创建一个 File 对象来标识这个文件，然后根据这个 File 对象创建真正读取文件的操作对象，这时将会真正创建一个关联真实存在的磁盘文件的文件描述符 FileDescriptor，这个这个对象可以操作磁盘文件。
由于我们需要的是字符格式，所以需要 StreamDecoder 类将 byte 解码为 char 格式。至于如何从磁盘上读取一段数据，操作系统会帮我们完成。而操作系统如何将数据持久化到磁盘以及如何创建数据结构的，需要根据操作系统使用何种文件系统来回答。

### java 序列化技术
Java 序列化是将一个对象转化成一串二进制表示的字节数组，通过保存或转移这些字节数据来达到持久化的目的。需要序列化，对象必须继承 java.io.Serializable 接口。反序列化则是相反的过程，将这个字节数组再重新构造成对象，反序列化必须有原始类作为模板，才能将这个对象还原。
**那么序列化后的信息包含哪些信息呢？来实际看下吧** 
```java
public class Serialize implements Serializable{

    private static final long serialVersionUID = -1687280615424697762L;

    public int num = 1390;

    public static void main(String... args) {
        try {
            FileOutputStream fos = new FileOutputStream("/Users/serialize.dat");
            try (ObjectOutputStream oos = new ObjectOutputStream(fos)) {
                Serialize serialize = new Serialize();
                oos.writeObject(serialize);
                oos.flush();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
输出如下：
```shell
aced 0005 7372 0015 636f 6d2e 6c69 616e
6a69 612e 5365 7269 616c 697a 65e8 9593
3849 355a 5e02 0001 4900 036e 756d 7870
0000 056e 
```
分为5部分（具体哪一段代表什么信息，还需要具体看）：
- 第一部分是序列化文件头
- 第二部分是序列化的类描述
- 第三部分是对象中各个属性项的描述
- 第四部分输出该对象的父类信息描述
- 第五部分输出对象的属性项的实际值

虽然 java 的序列化能保证对象状态的持久保存，但是遇到一些对象结构复杂的情况还是比较难处理的，下面是一些总结：
- 当父类继承了 Serializable 接口时，子类都可以被序列化
- 子类实现了 Serializable 接口，父类没有，则父类中的属性不能被序列化（不报错，数据丢失），但是子类属性可以正确序列化
- 如果序列化属性是对象，则这个对象也必须可以被序列化，否则报错
- 在反序列化是，如果对象的属性有修改或删减，则修改的部分属性会丢失，但不会报错
- 在反序列化是，如果 serialVersionUID 被修改，则反序列化会失败。（如果不显示声明，则会在编译时自动生成，但字段有修改后，自动生成的不一致，会导致失败，最好显式指定）

在纯java环境下，java 序列化能够很好的工作，但是在多语言的环境下，用java 序列化存储后，很难用其他语言还原出结果。在这种情况下，还是尽量存储通用的数据结构，如 JSON 或者 XML 数据结构。
