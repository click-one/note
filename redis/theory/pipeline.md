# 管道

## redis默认的请求方式

当我们使用客户端对 Redis 进行一次操作时，如下图所示，客户端将请求传送给服务器，服务器处理完毕后，再将响应回复给客户端。这要花费一个网络数据包来回的时间。 

![](../../.gitbook/assets/image%20%2827%29.png)

如果连续执行多条指令，那就会花费多个网络数据包来回的时间。如下图所示：

 

![](../../.gitbook/assets/image%20%2824%29.png)

客户端是经历了写-读-写-读四个操作才完整地执行了两条指令。 

![](../../.gitbook/assets/image%20%2823%29.png)

调整读写顺序，改成 写—写-读-读，这两个指令同样可以正常完成。 

![](../../.gitbook/assets/image%20%2826%29.png)

两个连续的写操作和两个连续的读操作总共只会花费一次网络来回，就好比连续的 write 操作合并了，连续的 read 操作也合并了一样。 

![](../../.gitbook/assets/image%20%2822%29.png)

这便是管道操作的作用，服务器根本没有任何区别对待，还是收到一条消息，执行一条消息，回复一条消息的正常的流程。客户端通过对管道中的指令列表改变读写顺序就可以大幅节省 IO 时间。在cpu极限范围内管道中指令越多，效果越好。 

## 压力测试

Redis 性能测试是通过同时执行多个命令实现的。（该命令是在 redis 的目录下执行的，而不是 redis 客户端的内部指令。）

```text
lvleideMacPro:bin lvlei$ redis-benchmark -t set -q
SET: 61842.92 requests per second
```

```text
lvleideMacPro:bin lvlei$ redis-benchmark -t set -P 2 -q
SET: 107765.09 requests per second
```

加上管道管道参数-p\(单个管道内并行的请求数量\)，qps有明显提示。

```text
lvleideMacPro:bin lvlei$ redis-benchmark -t set -P 150 -q
SET: 575284.12 requests per second
```

当把管道参数加到150左右以后，qps已经基本不变了，这个是因为redis的单线程cpu的使用率已经达到了100%，已经无法再提升。

## 管道的原理

![&#x5BA2;&#x6237;&#x7AEF;&#x548C;&#x670D;&#x52A1;&#x7AEF;&#x6570;&#x636E;&#x4EA4;&#x4E92;&#x8FC7;&#x7A0B;](../../.gitbook/assets/image%20%2829%29.png)

1. 客户端进程调用**write**将消息写到操作系统内核为套接字分配的发送缓冲**send buffer**。 
2. 客户端操作系统内核将发送缓冲的内容发送到网卡，网卡硬件将数据通过「网际路由」送到服务器的网卡。 
3. 服务器操作系统内核将网卡的数据放到内核为套接字分配的接收缓冲**recv buffer**。 
4. 服务器进程调用**read**从接收缓冲中取出消息进行处理。 
5. 服务器进程调用**write**将响应消息写到内核为套接字分配的发送缓冲**send** **buffer**。 
6. 服务器操作系统内核将发送缓冲的内容发送到网卡，网卡硬件将数据通过「网际路由」送到客户端的网卡。 
7. 客户端操作系统内核将网卡的数据放到内核为套接字分配的接收缓冲**recv** **buffer**。 
8. 客户端进程调用**read**从接收缓冲中取出消息返回给上层业务逻辑进行处理。 

**read操作：**write 操作只负责将数据写到本地操作系统内核的发送缓冲然后就返回了。剩下的事交给操作系统内核异步将数据送到目标机器。但是如果发送缓冲满了，那么就需要等待缓冲空出空闲空间来，这个就是写操作 IO 操作的真正耗时。

**read 操作：** read 操作只负责将数据从本地操作系统内核的接收缓冲中取出来就了事了。但是如果缓冲是空的，那么就需要等待数据到来，这个就是读操作 IO 操作的真正耗时。 

write操作几乎没有耗时，直接写到发送缓冲就返回，而read就会比较耗时了，因为它要等待消息经过网络路由到目标机器处理后的响应消息,再回送到当前的内核读缓冲才可以返回。这才是一个网络来回的真正开销。 

