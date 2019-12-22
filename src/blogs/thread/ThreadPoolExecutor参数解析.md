### 引子
线程池在项目中很常用，需要多个任务异步执行的地方我们都会去创建一个线程池。
我们看到 ```ThreadPoolExecutor```源码中提供了更方便的工厂方法（Executors）使用。
![](https://img2018.cnblogs.com/blog/839979/201903/839979-20190319104615347-50091239.png)
提供方便应该是更好的，而阿里针对线程池工厂方法的使用做了限制，是为什么呢？
![](https://img2018.cnblogs.com/blog/839979/201903/839979-20190319104632062-1588050765.png)
**限制的恰好是工厂方法中对应提供的几个方法，让我们带着疑问去看源码是为什么**

### 分析
ThreadPoolExecutor 提供了很多参数，分别介绍一下：
- corePoolSize 线程池中最少的工作线程，不允许销毁，除非设置了 allowCoreThreadTimeOut 参数
- maximumPoolSize 线程池中最多工作线程数（最大可以是2^29-1个）
- workQueue 任务队列，用于保存任务和执行任务之间切换
- keepAliveTime: 如果 当前线程数量 > corePoolSize，多出来的线程会在keepAliveTime之后就被释放掉
- unit: keepAliveTime的时间单位,比如分钟,小时等
- handler: 就是说当线程,队列都满了,之后采取的策略,比如抛出异常等策略

**接下来我们通过实际场景分析一下。**
我们假定有这个场景：

```
corePoolSize：1
mamximumPoolSize：3
keepAliveTime：60s
workQueue：ArrayBlockingQueue，有界阻塞队列，队列大小是4
handler：默认的策略，抛出来一个ThreadPoolRejectException
```

以下是新任务来时的场景：以poolSize=0 表示线程数量
- 来了一个任务，poolSize<corePoolSize 新建一个线程
- 又来了一个任务，poolSize>=corePoolSize ，队列未满，将任务丢入队列等待执行
- 继续添加任务，如果队列满了，而 poolSize < maximum，此时将会新建线程
- 如果继续添加任务，队列满，线程数量达到maximum，则会让handler 去处理。默认抛出异常
- 如果现在线程数量是3，但是都处于空闲状态，空闲超过60s 之后，其中2个线程就会被回收，保留一个（coolPoolSize=1）

所以当任务来临时的处理顺序是这样的：
- 首先创建 corePoolSize 线程
- 然后丢到队列等待
- 队列满，新建maximum线程
- 继续满，handler 处理

### 被禁止的原因
- 先看看 FixedThreadPool
  ![](https://img2018.cnblogs.com/blog/839979/201903/839979-20190319112956038-1349491972.png)
  可以看到，队列是没有限制大小的，所以，不会出现让 handler 去处理的情况。
  **SingleThreadPool 同理**
- 再看看CachedThreadPool
  ![](https://img2018.cnblogs.com/blog/839979/201903/839979-20190319113205106-2878652.png)
  可以看到，最大线程数是 Integer.MAX_VALUE ,线程数量太多。
  **ScheduledThreadPool 同理**

### 总结
**使用线程池前，先弄清楚各个参数的意义，然后再去使用自定义参数，让后续看代码的人可以清楚的知道线程的瓶颈，根据需要去使用。**