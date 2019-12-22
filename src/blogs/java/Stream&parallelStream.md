## 深入浅出 parallelStream
> ref: [深入浅出 parallelStream](https://blog.csdn.net/darrensty/article/details/79283146)
- 什么是流

Stream 是 java8 中新增加的一个特性，统称为流。
Stream 不是集合元素，也不是数据结构，不存储数据，是一个有关算法和计算，更像一个高级版本的迭代器Iterator。
用户只要给出需要对其包含的数据执行什么操作，比如“过滤”，“获取首字母”等，Stream 会隐式的在内部进行遍历，做出相应的数据转换。

Stream 就如同一个迭代器（Iterator），单向、不可往复，数据只能遍历一次，遍历过一次后即用尽了，就好比流水从面前流过。

Stream 可以并行化操作数据，数据分为多段，每一个都在不同的线程中处理。然后将结果一起输出。

Stream 的并行操作依赖于 java 7 中引入的 Fork/join 框架来拆分任务和加速处理过程。

### parallelStream 
parallelStream 其实就是一个并行执行的流，它通过默认的 ForkJoinPool，可能提高你的多线程任务速度。

- parallelStream 的作用
并行处理任务，将一个大任务切分成多个小任务，这表示每一个任务都是一个操作。因此，如下代码段输出的可能不是顺序输出：
```java
List<Integer> numbers = Arrays.asList(1,2,3,4);
numbers.parallelStream().forEach(out::println);
```
如果希望顺序输出，也可以使用 ```forEachOrdered```方法（但可能会失去并行的优势）。

### forkJoinPool
从java 7 中引入，同 ThreadPoolExecutor 一样，实现了 Executor 和 ExecutorException 接口。它使用了一个无限队列来保存需要执行的任务，而线程的数量则是通过构造函数传入，如果没有向构造函数中传入希望的线程，那么当前计算机可用的CPU数量会被设置为线程数量作为默认值。

ForkJoinPool 主要使用分治法来解决问题。主要的要点是，它使用有线的线程来处理大量的任务。也就是一个线程可以处理一个任务，与 ThreadPoolExecutor 不同的是，这个任务可以有多个子任务，可以等待子任务完成之后，在去处理父任务。

- 工作窃取算法
就是利用现代设备多核，在一个操作时候会有空闲的CPU，那么如何利用好这个空闲的CPU 就成了提高性能的关键，而这里我们提到的工作窃取（work-steal）算法就是整个forkjoinPool 的核心。
工作窃取算法就是值某个线程从其他队列里窃取任务来执行。

**那么为什么需要使用工作窃取算法呢？**
假如我们需要做一个很大的任务，我们可以吧这个任务分割为若干个互不依赖的子任务，为了减小线程间的竞争，于是把这些子任务分别
放在不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和任务一一对应，比如A线程负责处理A队列
里的任务。但是有的线程会把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如
去帮其他线程干活，于是就去其他线程的队列里获取任务来执行。而这时他们会访问同一个队列，所以为了减小窃取任务线程和被窃取任务线程之间的竞争，
通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

> 工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

java 8 为 forkJoinpool 添加了一个通用的线程池，这个线程池用来处理那些没有被显示提交到任务线程池的任务。
他是 ForkJoinPool 类型上的一个静态元素，他拥有的默认线程数量等于运行计算机上的处理器数量。当调用Arrays类上
添加的新方法时，自动并行化就会发生。自动并行化也会运用于java8 新添加的Stream API 中。

```java
    List<UserInfo> userInfoList =
        DaoContainers.getUserInfoDAO().queryAllByList(new UserInfoModel());
    userInfoList.parallelStream().forEach(RedisUserApi::setUserIdUserInfo);
```
对于以上代码，列表中的元素会并行进行操作，forEach 方法会为每个元素的计算操作创建一个任务，该任务会被 ForkJoinPool 中的线程处理。
以上代码当然可以使用 ThreadPoolExecutor 处理，但是就可读性和代码量而言，ForkJoinPool 明显更胜一筹。

对于 ForkJoinPool 的线程池的大小一般使用默认值，也可以通过 -Djava.util.concurrent.ForJoinPool.common.parallelism=N
来设置大小。

经过测试，ForkJoinPool 的线程池是公用的，所以一个项目中只有一个线程池可用。而且，forEach 方法也会生成一个线程。

**总结：
1、当需要处理递归分治算法时，考虑使用 ForkJoinPool\
2、仔细设置不再进行任务划分的阈值，这个阈值对性能有影响\
3、Java 8 中的一些特性会使用到ForkJoinPool中的通用线程池，在某些场景下需要修改默认线程数。**

### 要点
1、因为parallelStream 使用的是一个公用的线程池。
2、如果有其他地方使用了parallelStream，并且方法是一个耗时的方法，此时，common 中的 线程池会被很快耗尽。任务在队列中排队时会被阻塞。
成为阻塞程序的源头。
3、所以如果一定需要多线程执行，可以单独配置线程池去执行。


