# 

## CMD跟ENTRYPOINT优先级

- 准备两份Dockerfile

```dockerfile
FROM alpine:3.2
CMD echo "cmd"
ENTRYPOINT echo "enter"
```

```dockerfile
FROM alpine:3.2
ENTRYPOINT echo "enter"
CMD echo "cmd"
```

> 两次输出结论都是enter,结合官网的文档说明可以看出ENTRYPOINT的优先级高于CMD

## CMD或者ENTRYPOINT使用

- 只使用cmd命令的时候

```sh
docker run temp:1 替换命令(可选)、

docker run temp:1 echo "替换"
```

- 只使用entrypoint的时候

```
docker run --entrpoint=替换命令 镜像
```

## CMD与ENTRYPOINT交叉使用

> 使用exec的方式搭配使用例如以下的实际命令是 /usr/sbin/nginx  -h

```dockerfile
ENTRYPOINT ["/usr/sbin/nginx"]

CMD ["-h"]
```

- 更改

```
docker run .... temp:1 -g"daemon off;"
```

> 实际命令是 /usr/sbin/nginx  -g"daemon off;"