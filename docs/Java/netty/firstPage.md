Netty可以选择IO模式,可以无缝切换
## IO模型切换
Common|Linux|Mac
---|---|---|
NioEventLoopGroup|	EpollEventLoopGroup|	KQueueEventLoopGroup
NioEventLoop|	EpollEventLoop|	KQueueEventLoop
NioServerSocketChannel|	EpollServerSocketChannel	|KQueueServerSocketChannel
NioSocketChannel|	EpollSocketChannel	|KQueueSocketChannel


## 水平模式/边缘模式
> JDK 的 NIO 在 Linux 上默认是**水平触发（LT）**；Netty 的 native epoll transport 默认是**边缘触发（ET）**，可以通过 `EpollChannelOption.EPOLL_MODE` 设为 `EpollMode.LEVEL_TRIGGERED` 切回水平触发。
### 水平模式
当epoll_wait函数检测到有事件发生并将通知应用程序，而应用程序不一定必须立即进行处理，这样epoll_wait函数再次检测到此事件的时候还会通知应用程序，直到事件被处理。

### 边缘模式
只要epoll_wait函数检测到事件发生，通知应用程序立即进行处理，后续的epoll_wait函数将不再检测此事件。因此ET模式在很大程度上降低了同一个事件被epoll触发的次数

### 两种模式怎么选
- 水平触发（LT）：只要 fd 还就绪，`epoll_wait` 就会反复通知，直到你把它读完。编程简单、不容易漏事件，但可能重复通知。
- 边缘触发（ET）：状态变化时只通知一次，必须一次把数据读干净（通常配合非阻塞 + 读到 EAGAIN），否则剩余数据可能再也收不到通知。效率更高，但实现要求更严格。

JDK NIO 在 Linux 上用 LT；Netty 的 native epoll 默认用 ET（性能优先），需要时可通过 `EpollChannelOption.EPOLL_MODE` 设为 `EpollMode.LEVEL_TRIGGERED`。


