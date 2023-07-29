## 最近遇到一个奇怪的问题？
> 基于netty实现的websocket,收到的字节长度不对。发送字符长度为3M,只收到了1.7M的数据,导致业务在json解析的时候
出现异常

## 如何解决这个问题？
- 数据是如何流转？
> 域名->nginx->后端服务(SpringBoot)  
- nginx配置的websocket代理是否正确？
```shell
    location / {
        proxy_pass   http://127.0.0.1:8080/; // 代理转发地址
 　　　　proxy_http_version 1.1;
        proxy_read_timeout   3600s; // 超时设置
        // 启用支持websocket连接
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
      }
```  
- nginx是否截断了大文本的数据?
> nginx使用gzip压缩数据,这个时候根据日志对size的计算就不精准。使用抓包工具tcpdump,抓包分析

> 1 nginx收到的包 2后端服务收到的包。

> 确认两端收到包的size无误,没有出现截断的情况发生
- netty是如何读取数据？
> 读了百度能搜到大部分的文章,结合源码debug。判断收到的数据是完整的
推荐文章：https://zhuanlan.zhihu.com/p/494916509

然后整个人懵逼了,what fuck?

这个时候掉转枪头,然后怀疑自己代码问题。自定义decoder的时候,没有正确处理收到的数据。
按照这个思路,去google立刻就得到了答案。

分片的数据,还需要结合ContinuationWebSocketFrame使用。

参考伪代码如下
```shell
//正常业务数据
if (frame instanceof TextWebSocketFrame) {
   frameBuffer = new StringBuilder();
   TextWebSocketFrame textFrame = (TextWebSocketFrame) frame;
   String text = textFrame.text();
   frameBuffer.append(text);
} else if (frame instanceof ContinuationWebSocketFrame) {
    ContinuationWebSocketFrame continuationWebSocketFrame = (ContinuationWebSocketFrame) frame;
    String cwsText = continuationWebSocketFrame.text();
    frameBuffer.append(cwsText);
} else {
   return;
}
if (frame.isFinalFragment()) {
    handleMessageCompleted(ctx, frameBuffer.toString());
 }
```








