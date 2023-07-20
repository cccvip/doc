Netty可以选择IO模式,可以无缝切换
## IO模型切换
Common|Linux|Mac
---|---|---|
NioEventLoopGroup|	EpollEventLoopGroup|	KQueueEventLoopGroup
NioEventLoop|	EpollEventLoop|	KQueueEventLoop
NioServerSocketChannel|	EpollServerSocketChannel	|KQueueServerSocketChannel
NioSocketChannel|	EpollSocketChannel	|KQueueSocketChannel


## 水平模式/边缘模式
> JDK的 NIO 默认实现是水平触发，Netty 是边缘触发和水平触发(默认)可切换
### 水平模式
当epoll_wait函数检测到有事件发生并将通知应用程序，而应用程序不一定必须立即进行处理，这样epoll_wait函数再次检测到此事件的时候还会通知应用程序，直到事件被处理。

### 边缘模式
只要epoll_wait函数检测到事件发生，通知应用程序立即进行处理，后续的epoll_wait函数将不再检测此事件。因此ET模式在很大程度上降低了同一个事件被epoll触发的次数

### 为啥默认是水平触发
内核告诉应用程序一个文件描述符是否就绪了，然后可以对就绪的fd进行IO操作。如果不作任何操作，内核还是会继续通知。


