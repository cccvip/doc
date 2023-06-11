> Boss EventloopGroup 和 worker EventloopGroup 分别 处理 接收连接， 和读写。

注意点：
- Eventloopsize 建议设置为 2 的次方，dispatch 使用位移，更快。
- 侦听一个端口，只会绑定到 BossEventLoopGroup 中的一个 Eventloop，所以， BossEventLoopGroup
配置多个也无用。

[参考文章](https://www.cnblogs.com/flydean/p/15963985.html)


EventLoop
如果只使用 tcp 和 异步阻塞的话主要关心以下2个 EventLoop

- NioEventLoop - 基于java 原生nio

- EpollEventLoop - native jni 直接调用 epoll(only work on linux)