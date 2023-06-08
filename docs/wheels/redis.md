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

#### Demo

参考代码

#### I/O多路复用
通过对Redis的源码解读中发现,redis是一个单线程的server,通过执行select函数,select函数会阻塞，
直到有描述符就绪（有数据可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。

源码地址：ae.c
```shell
     //核心代码 执行select()函数 会造成阻塞的方式
    retval = select(maxfd+1, &rfds, &wfds, &efds, tvp);
```
是不是很惊喜,很意外,怎么跟说好的不一样呢? 常常背诵的八股文中,提到了我们使用的是单线程,使用IO多路复用等技术处理呢。























































