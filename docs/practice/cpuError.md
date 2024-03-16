## cpu负载跟利用率

CPU使用率打到100%
1 通过top找到占用率最高的进程
2 通过top -Hp pid 找到占用最高的线程ID
3 再把线程ID转化为16进制，printf "0x%x\n" 958，得到线程ID0x3be
4 jstack 进程ID | grep '0x3be' -C5 --color
