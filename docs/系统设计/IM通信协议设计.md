## 评估标准
1. 性能 消息协议效率问题,尽可能的较低端到端的延迟
2. 兼容 系统版本的更新,向前向后兼容
3. 存储 消息包尽可能小,降低存储
4. 计算 编解码的问题
5. 网络 尽可能的快速传输
6. 安全 协议安全性,防止协议被破解
7. 迭代 灵活扩展,方便IM系统的迭代开发
8. 通用 可跨平台的接入H5,客户端,IOT设备

基本结构
应用层
使用PB作为传输数据,数据压缩率较高
安全层

参考文章地址
[IM消息时序一致性](http://www.52im.net/thread-3189-1-1.html)

[TCP的长连接和短连接](https://developer.aliyun.com/article/37987)

[理解IM消息的可靠性和一致性问题,以及解决方案](https://cloud.tencent.com/developer/article/1833287)

[保证在线实时消息的可靠投递](http://www.52im.net/thread-294-1-1.html)

[IM消息时序性和一致性](https://zhuanlan.zhihu.com/p/462668553)

[微信的海量IM聊天消息序列号生成实践](http://www.52im.net/thread-1998-1-1.html)

[微信的海量IM聊天消息序列号生成实践](http://www.52im.net/thread-1999-1-1.html)
