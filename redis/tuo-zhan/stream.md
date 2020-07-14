# Stream

![](../../.gitbook/assets/image%20%2832%29.png)

Redis Stream 的结构如上图所示，它有一个消息链表，将所有加入的消息都串起来，每个消息都有一个唯一的 ID 和对应的内容。消息是持久化的，Redis 重启后，内容还在。 

每个 Stream 都有唯一的名称，它就是 Redis 的 key，在我们首次使用xadd指令追加消息时自动创建。 

每个 Stream 都可以挂多个消费组，每个消费组会有个游标last\_delivered\_id在 Stream 数组之上往前移动，表示当前消费组已经消费到哪条消息了。每个消费组都有一个 Stream 内唯一的名称，消费组不会自动创建，它需要单独的指令xgroup create进行创建，需要指定从 Stream 的某个消息 ID 开始消费，这个 ID 用来初始化last\_delivered\_id变量。 

每个消费组 \(Consumer Group\) 的状态都是独立的，相互不受影响。也就是说同一份 Stream 内部的消息会被每个消费组都消费到。 

同一个消费组 \(Consumer Group\) 可以挂接多个消费者 \(Consumer\)，这些消费者之间是竞争关系，任意一个消费者读取了消息都会使游标last\_delivered\_id往前移动。每个消费者有一个组内唯一名称。 

消费者 \(Consumer\) 内部会有个状态变量pending\_ids，它记录了当前已经被客户端读取的消息，但是还没有 ack。如果客户端没有 ack，这个变量里面的消息 ID 会越来越多，一旦某个消息被 ack，它就开始减少。这个 pending\_ids 变量在 Redis 官方被称之为PEL，也就是Pending Entries List，这是一个很核心的数据结构，它用来确保客户端至少消费了消息一次，而不会在网络传输的中途丢失了没处理。 

## 消息ID

消息 ID 的形式是timestampInMillis-sequence，例如1527846880572-5，它表示当前的消息在毫米时间戳1527846880572时产生，并且是该毫秒内产生的第 5 条消息。消息 ID 可以由服务器自动生成，也可以由客户端自己指定，但是形式必须是整数-整数，而且必须是后面加入的消息的 ID 要大于前面的消息 ID。 

## 增删改查

* xadd 追加消息 
* xdel 删除消息，这里的删除仅仅是设置了标志位，不影响消息总长度 
* xrange 获取消息列表，会自动过滤已经删除的消息 
* xlen 消息长度 
* del 删除 Stream

## 消费者组

![](../../.gitbook/assets/image%20%2833%29.png)

```text
# 表示从头开始消费
127.0.0.1:6379> xgroup create codehole cg1 0-0
OK
# $ 表示从尾部开始消费，只接受新消息，当前 Stream 消息会全部忽略
127.0.0.1:6379> xgroup create codehole cg2 $
OK
```

## **Stream** 消息太多怎么办？

要是消息积累太多，Stream 的链表岂不是很长，内容会不会爆掉?xdel 指令又不会删除消息，它只是给消息做了个标志位。 所以它提供了一个定长 Stream 功能。在 xadd 的指令提供一个定长长度 maxlen，就可以将老的消息干掉，确保最多不超过指定长度。 

## PEL如何避免消息丢失

在客户端消费者读取 Stream 消息时，Redis 服务器将消息回复给客户端的过程中，客户端突然断开了连接，消息就丢失了。但是 PEL 里已经保存了发出去的消息 ID。待客户端重新连上之后，可以再次收到 PEL 中的消息 ID 列表。

## Stream的高可用

Stream 的高可用是建立主从复制基础上的，它和其它数据结构的复制机制没有区别，也就是说在 Sentinel 和 Cluster 集群环境下 Stream 是可以支持高可用的。不过鉴于 Redis 的指令复制是异步的，在 failover 发生时，Redis 可能会丢失极小部分数据。





