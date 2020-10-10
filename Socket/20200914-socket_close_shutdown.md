# Socket 之Close与Shutdown

## 区别

1. close会释放文件句柄，而shutdown不会。
2. close把描述符的引用计数减1，仅在该计数变为0时关闭套接字。shutdown可以不管引用计数就激发TCP的正常连接终止序列，因此在多进程环境中：close()是关闭本进程的socket id，但链接还是开着的，用这个socket id的其它进程还能用这个链接，能读或写这个socket id，而shutdown执行的操作对所有进程有效。

## Strace

```shell
strace ./exec
```

