从0开始,就要从redis的远古时期的版本开始仿写,代码简单,新人友好。参考B站UP（楚国刮大风），阅读的redis的源码部分。
源码下载地址(https://download.redis.io/releases/)，redis-server版本为0.091。

参考书籍 《Redis设计与实现》 《Redis开发与运维》

首先需要了解关于redis几个重要特征

- redis-server工作原理,工作流程

- 使用了多少种数据结构

- 事件驱动模式

- 集群实现

## redis的server交互简易流程图

源码从0开始看,同理先从网络编程socket的几个基本概念开始说。

客户端与服务器端的交互简易流程图如下

![server.png](redisImage/server.png)

> 基础版本的redis,所有的操作都是单线程。

- client向server注册的事件为read事件
  > 只需要server有数据返回,client端就可以立即得到数据
- server完成处理后、向client注册write事件
  > client能收到server端的数据
- 通信完client关闭
  > client.close()
- 流程结束

### 学习掌握Nio网络编程

参考文章 [https://itimetraveler.github.io/2018/05/15/%E3%80%90Java%E3%80%91NIO%E7%9A%84%E7%90%86%E8%A7%A3/#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90]

要学习并且掌握Nio編程,首先要理解几个概念作为基础条件

#### 内核态/用户态

非底层语言的代码,程序只能在用户态的进程中进行操作.(性能损失)

#### 文件描述符fd

在unix世界里,一切进程都是文件。fd就是一个索引下标,举个例子：如果需要操作文件,只需要通过fd索引到具体文件,则可以执行CRUD操作. fd在redis的源码中处处可见。

- *那么在java中fd在哪儿呢？*

```shell
 ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
 源码如下
 ServerSocketChannelImpl(SelectorProvider var1) throws IOException {
        super(var1);
        //返回fd 描述文件
        this.fd = Net.serverSocket(true);
        this.fdVal = IOUtil.fdVal(this.fd);
        this.state = 0;
    }
```

#### I/O多路复用

> I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回。

执行select函数,select函数会阻塞， 直到有描述符就绪（有数据可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。

源码0.0.91v 源码地址：ae.c

```shell
     //核心代码 执行select()函数 会造成阻塞的方式
    retval = select(maxfd+1, &rfds, &wfds, &efds, tvp);
```

源码1.3.6v中
> ae.c 323行

```shell
  //使用epoll技术
numevents = aeApiPoll(eventLoop, tvp);
```

#### Epoll vs Select vs Poll

Select
> 当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

> select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select的一 个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但 是这样也会造成效率的降低。

Epoll的优势
> 在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一 个文件描述符，
> 一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。)

epoll的优点主要是一下几个方面：

- 监视的描述符数量不受限制

> 它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。select的最大缺点就是进程打开的fd是有数量限制的。这对 于连接数量比较大的服务器来说根本不能满足。虽然也可以选择多进程的解决方案( Apache就是这样实现的)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。

- IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。

> 如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。

#### 框架选型

原生的Nio编程也是可以的,因为jdk在1.5之后,也提供了epoll的使用,加载方式使用SPI机制

> 为了更方便,更容易使用NIO编程,首选Netty框架

Let us coding!

## 数据结构
> java已经实现了基础数据结构类型,例如如下类型
- String
- Set
- SortedSet
- Map

## 命令行解析
命令行[https://www.redis.net.cn/order/]

## 传输协议



















