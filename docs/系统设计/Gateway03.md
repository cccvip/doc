# 网关协议转换

在开发联调中,如果有一个接口只暴露Rpc接口调用方式,但是为提供给其他环境使用需要以Http形式暴露接口。此时就涉及到协议转换,

- Http的请求->Rpc请求。

- Http的请求->Http请求。

如何实现呢?

## 第一种Http->Rpc

Rpc框架选择Dubbo,开源Rpc框架活跃度第一。

思考方式转换开始

- 第一步: 网关扮演的角色

> 我们都知道Rpc框架有三个重要的角色,接口生产者,接口消费者,注册中心。此刻网关的角色不主动实现接口逻辑,那么网关就是接口消费者。

- 第二步: 网关如何调用Rpc接口

> Rpc框架的重要特性之一,泛化调用(使用可参考[dubbo泛化调用](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/advanced-features-and-usage/service/generic-reference/))

- 第三步: 网关如何返回数据

> Netty提供的FullHttpResponse对象,只需要封装数据返回即可

## 第二种Http->Http

Http调用Http需要再细分两种情况,服务内以及服务外

### 注册接口在网关服务内

具体实现可参考OpenFeign

### 注册接口在网关服务外
可使用okhttp,RestTemplate等模拟请求调用。























