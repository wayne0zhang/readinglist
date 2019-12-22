## 1. 错误
今天项目中出现了大量的`java.net.ConnectException: Cannot assign requested address (connect failed) `
错误。

刚开始以为是服务提供方的服务器报出来的错误，还和对方怼了几句。但是之后在网上搜索之后发现，这个是客户端机器的问题：

这个错误的报出是由于客户端服务器的出口`socket`端口被占用完了，没有办法对新的请求进行分配端口，就会出现这种错误。

排查当时的情况，确实有大量的依赖服务调用出现。

但是这种情况不是每天都有，而是偶发性的。所以和我们自己的服务有关系，由于业务原因导致了端口被占用完了。

## 2. 原因
google 之后发现了一个大致的原因：建立TCP连接之后，断开连接的一方会进入 `TimeWait`状态，此时这个连接是不可以
复用的。而连接是由4元组（localIP/port,serverIp/port）来唯一确定一个连接，对于一个客户端来说，server端的都确定了（单台服务情况），
而本机ip不会变化，所以能变化的只有服务器提供的临时端口。

我们知道端口是由2个字节描述的，也就是范围是65525.再加上服务器有一些默认端口被占用，可供临时分配的端口大概会在2w+左右[ref](https://www.cnblogs.com/duanxz/p/4464178.html)。

当短时间大量请求出现的时候，端口是有占满的可能，此时就会报错误，[ref](https://blog.csdn.net/test1280/article/details/80295435).

既然请求都结束了，为什么还需要占用端口呢，有没有办法复用这些端口呢？

## 3. 一种解决途径
答案是肯定的，unix 有一个参数 `SO_REUSEADDR`设置 处于 `Time_wait`状态的端口可以被复用，如下是翻译：

> 这个socket 参数是用来告诉内核，当一个端口处于Time_wait 状态的时候,依然可以使用这个端口。但是如果端口不是这个状态，那么还是会
得到一个错误：地址被占用。这个参数可以很方便的使用在很快结束并又重新建立连接的服务。
但是你需要知道，如果有其他不可预知的数据进来时，会破坏你的服务。但是这个可能性很低。

> Michael Hunter (mphunter@qnx.com) 指出过：网络5元组用于确定连接唯一性。这个参数仅仅说明你可以复用本地的地址。连接还需要保证
网络连接的唯一性。也就是说如果你访问的目标服务器还是同一台，那么连接还会是同一个；如果不是同一台，那么不是同一个连接。

> 危险仅出现在当一个断开的连接还是在使用中，同一台客户端调用了同一个服务端，此时就会出现使用同一个连接的情况。也就得到意想不到的数据。

> 这也就是为什么有 `Time_wait` 状态出现的原因，为了保证短时间内连接不可复用。


> [原文如下](http://www.unixguide.net/network/socketfaq/4.5.shtml)
  What exactly does SO_REUSEADDR do?
  This socket option tells the kernel that even if this port is busy (in
  the TIME_WAIT state), go ahead and reuse it anyway.  If it is busy,
  but with another state, you will still get an address already in use
  error.  It is useful if your server has been shut down, and then
  restarted right away while sockets are still active on its port.  You
  should be aware that if any unexpected data comes in, it may confuse
  your server, but while this is possible, it is not likely.
\
  It has been pointed out that "A socket is a 5 tuple (proto, local
  addr, local port, remote addr, remote port).  SO_REUSEADDR just says
  that you can reuse local addresses.  The 5 tuple still must be
  unique!" by Michael Hunter (mphunter@qnx.com).  This is true, and this
  is why it is very unlikely that unexpected data will ever be seen by
  your server.  The danger is that such a 5 tuple is still floating
  around on the net, and while it is bouncing around, a new connection
  from the same client, on the same system, happens to get the same
  remote port.  This is explained by Richard Stevens in ``2.7 Please
  explain the TIME_WAIT state.''.
  
## 4. 其他
mysql连接池技术就是为了复用连接，减少TCP握手挥手的开销而做的，所以使用连接池技术后，连接数应该不会很大。

而kafka的TCP连接数和broker数量有关系，大概是 2*n 的关系，所以kafka 不会有很多的TCP连接产生。
[关于Kafka java consumer管理TCP连接的讨论](https://www.cnblogs.com/huxi2b/p/10215956.html)