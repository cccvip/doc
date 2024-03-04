## 吐槽
SCG的性能损耗太严重,二次开发有多难,相信大家都深有感触。但是技术架构是团队选择,非个人选择。遇到了只能硬着头皮上去尝试解决问题

## 第一个问题
线上的生产环境
```shell
nginx->scg->server
此时后端服务出现ClientAbortException,SCG超时,前端页面报错
2021-02-08 19:30:43.835  WARN 87602 --- [o-8080-exec-122] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.http.converter.HttpMessageNotReadableException: I/O error while reading input message; nested exception is org.apache.catalina.connector.ClientAbortException: java.io.EOFException: Unexpected EOF read on the socket]
```

线上生产环境出现的bug,可在github上找到对应提的issue。
[https://github.com/spring-cloud/spring-cloud-gateway/issues/2141](https://github.com/spring-cloud/spring-cloud-gateway/issues/2141)

分析过程简单描述一下,说一下我的排查思路
### 第一次排查
> google查询,看看有没有人出现过类似的问题,希望能从前辈的试错中找到解决办法。结果都是模棱两可的解释
- 升级reactor-netty从版本0.9.15升级到高版本0.9.25.release版本,实测无效,只能再花时间处理

### 第二次排查
- 修改SCG打印日志级别为Trace
- 抓包分析nginx->SCG的过程
- 抓包分析SCG->后端服务
> 通过对日志等分析,大概猜出问题点应该是SCG框架的问题,于是在github上找到有人曾经提出过的问题。但是没有解决办法

### 第三次排查
> 通过前两次的观察发现,connection被异常关闭,怀疑scg的连接池配置存在问题。

> 开始漫漫google过程,梳理SCG的执行流程,连接池的创建关闭过程

### 第四次排查
> 有些许想法后,开始修改配置文件将reactor.netty的连接idle-timeout设置为10s,将连接使用方式修改为lifo
```shell
/**
 * Default leasing strategy (fifo, lifo), fallback to fifo.
 * <ul>
 *     <li>fifo - The connection selection is first in, first out</li>
 *     <li>lifo - The connection selection is last in, first out</li>
 * </ul>
 * <p><strong>Note:</strong> This configuration is not applicable for {@link reactor.netty.tcp.TcpClient}.
 * A TCP connection is always closed and never returned to the pool.
 */
public static final String POOL_LEASING_STRATEGY = "reactor.netty.pool.leasingStrategy";

/**
 * Default max idle time, fallback - max idle time is not specified.
 * <p><strong>Note:</strong> This configuration is not applicable for {@link reactor.netty.tcp.TcpClient}.
 * A TCP connection is always closed and never returned to the pool.
 */
public static final String POOL_MAX_IDLE_TIME = "reactor.netty.pool.maxIdleTime";
```

### 验证
短期压测后,发现服务正常。


参考文章
- [https://projectreactor.io/docs/netty/snapshot/reference/index.html](https://projectreactor.io/docs/netty/snapshot/reference/index.html)
- [https://www.cnblogs.com/caicz/p/13963503.html](https://www.cnblogs.com/caicz/p/13963503.html)
- [https://peiqianggao.github.io/2020/08/17/SpringCloudGateway%E6%8A%A5%E9%94%99Connection-prematurely-closed-BEFORE-response/](https://peiqianggao.github.io/2020/08/17/SpringCloudGateway%E6%8A%A5%E9%94%99Connection-prematurely-closed-BEFORE-response/)

















