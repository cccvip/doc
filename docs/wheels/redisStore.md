# Redis持久化策略

> 低版本中没有持久化策略的实现
参考源码版本为
```shell
# buffer->disk
void flushAppendOnlyFile(void);
# write buffer
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc);
# 临时文件移除
void aofRemoveTempFile(pid_t childpid);
int rewriteAppendOnlyFileBackground(void);
# 初始化加载appendOnlyFile
int loadAppendOnlyFile(char *filename);
# 停止
void stopAppendOnly(void);
# 开始
int startAppendOnly(void);
# 守护进程,用于处理append file
void backgroundRewriteDoneHandler(int statloc);
```

## RDB

> 生成快照,进行存储

## AOF

> 每次修改后的数据,进行存储

## RDBvsAOF

> 技术哪有谁比谁强呢,强强联合才是最合适的。RDB周期存储+AOF轮询存储, 是最好的搭配组合。

## 如何实现？

- 写命令：(先写缓存,再写文件)

> 内容是RESP协议的文本格式,个人猜测可能是因为这样交互成本少一些

- 文件压缩: AOF文件会无限制的增长

> 

- 重启加载:命令重放(redis重启后,需要将写入的数据,存放到内存中)

> 













