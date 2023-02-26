
参考 [https://blog.csdn.net/yangkai_hudong/article/details/47783487](https://blog.csdn.net/yangkai_hudong/article/details/47783487)

## 查找日志通过关键字查找

- 一般使用
  
  ```
  tail -f run.log 实时输出日志
  tail 从尾部打印日志
  head 头部打印日志
  head 
  ```
  
  ```
  cat -n run.log |grep "关键字"  得到日志的行号
  cat -n run.log |tail -n +行号|head -n 自定义当前位置多少条
  ```
- 已知行号200，需要找200后100条记录
  
  ```
  cat -n run.log |tail -n +200|head -n 100
  ```


## 拷贝文件命令
```shell script
\cp xxx xxx
```
> 当拷贝大量文件的时候,会出现提示确认。尝试使用这种方法可以解决
