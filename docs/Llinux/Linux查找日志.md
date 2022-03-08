# Linux查找日志

参考 [https://blog.csdn.net/yangkai_hudong/article/details/47783487](https://blog.csdn.net/yangkai_hudong/article/details/47783487)

# 通过关键字查找

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


