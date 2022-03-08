## frpc浅尝

### 准备趁手的兵器

- 远程公网的服务器(带独立IP)
- 安装frpc
- 下载地址https://github.com/fatedier/frp/releases

安装

> 默认大家使用的服务器都是liunx,这个时候下载frp只需要对应系统架构下载合适的即可 
> 
> hostnamectl

```
大概如下：
  Static hostname: xx
Transient hostname: XX
         Icon name: xx
           Chassis: vm
        Machine ID: XX
           Boot ID: xx
    Virtualization: xx
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-957.1.3.el7.x86_64
      Architecture: x86-64
```

#### 配置frp服务端

- 修改frps.ini文件
  
  ```
  [common]
  bind_addr = 0.0.0.0
  bind_port = 7000
  privilege_token = XX
  ```

#### 配置frp客户端

- frpc.ini文件

```
[common]
server_addr = 服务端IP
server_port = 7000  与服务器配置一样
privilege_token = XX


[http]
type = tcp
local_port = 8080  本地服务端口
local_ip = 127.0.0.1
remote_port = 8080 客户端端口
```

### 调用

```shell
curl http://远程服务器端口:客户端端口  
```
