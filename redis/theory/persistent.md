# 持久化

redis中的所有数据都是存储在内存中，现在很多场景的时候，不只单单作为缓存来使用，很多时候直接作为数据库来使用。如果redis一旦宕机重启，内存中的所有数据都将丢失。为了避免redis提供了持久化的功能，RDB和AOF。

## RDB\(快照\)

RDB是一种快照存储持久化方式，具体就是将Redis某一时刻的内存数据保存到硬盘的文件当中，默认保存的文件名为dump.rdb，而在Redis服务器启动时，会重新加载dump.rdb文件的数据到内存当中恢复数据。

### save

```text
127.0.0.1:6379> save
OK
```

同步操作，会阻塞Redis。当客户端向服务器发送save命令请求进行持久化时，服务器会阻塞save命令之后的其他客户端的请求，直到数据同步完成。

### bgsave

```text
127.0.0.1:6379> bgsave
Background saving started
```

则fork 出一个子进程，子进程负责调用 rdbSave ，并在保存完成之后向主进程发送信号，通知保存已完成。因为 rdbSave 在子进程被调用，所以 Redis 服务器在 gbsave 执行期间仍然可以继续处理客户端的请求。

### redis服务端自动触发

除了通过客户端发送命令外，还有一种方式，就是在Redis配置文件中的save指定到达触发RDB持久化的条件，比如【多少秒内至少达到多少写操作】就开启RDB数据同步。 我们可以在配置文件redis.conf指定如下的选项：

```text
# 900s内至少达到一条写命令
save 900 1
# 300s内至少达至10条写命令
save 300 10
# 60s内至少达到10000条写命令
save 60 10000
```

## AOF

AOF持久化方式会记录客户端对服务器的每一次写操作命令，并将这些写操作以Redis协议追加保存到以后缀为aof文件末尾，在Redis服务器重启时，会加载并运行aof文件的命令，以达到恢复数据的目的。

### 写入方式

AOF 日志是以文件的形式存在的，当程序对 AOF 日志文件进行写操作时，实际上是将内容写到了内核为文件描述符分配的一个内存缓存中，然后内核会异步将脏数据刷回到磁盘的。提供了三种写入的方式 

* always。Redis的每条写命令都写入到系统缓冲区，然后每条写命令都使用fsync“写入”硬盘，效率很慢。
*  everysec。过程与always相同，只是fsync的频率为1秒钟一次。这个是Redis默认配置，如果系统宕机，会丢失一秒左右的数据 
* no。由操作系统决定什么时候从系统缓冲区刷新到硬盘，容易丢失数据。

### 文件重写

AOF将客户端的每一个写操作都追加到aof文件末尾，比如对一个key多次执行incr命令，这时候，aof保存每一次命令到aof文件中，aof文件会变得非常大。

```text
incr num 1
incr num 2
incr num 3
incr num 4
incr num 5
incr num 6
...
incr num 100000
```

aof文件太大，加载aof文件恢复数据时，就会非常慢，为了解决这个问题，Redis支持aof文件重写，通过重写aof，可以生成一个恢复当前数据的最少命令集，比如上面的例子中那么多条命令，可以重写为：

```text
incr num 100000
```

Redis 提供了 bgrewriteaof 指令用于对 AOF 日志进行瘦身。其原理就是开辟一个子进程对内存进行遍历转换成一系列 Redis 的操作指令，序列化到一个新的 AOF 日志文件中。序列化完毕后再将操作期间发生的增量 AOF 日志追加到这个新的 AOF 日志文件中，追加完毕后就立即替代旧的 AOF 日志文件

```text
127.0.0.1:6379> bgrewriteaof
Background append only file rewriting started
```

**文件重写的优点**

* 压缩aof文件，减少磁盘占用量。
* 将aof的命令压缩为最小命令集，加快了数据恢复的速度。

## 混合持久化

通过以上可以发现两种不同的持久化方式都会有相应的缺点：

* RDB持久化：如果服务器宕机会导致执行快照后的这一段时间内的数据丢失。
* AOF持久化：AOF方式生成的日志文件太大，即使通过AFO重写，文件体积仍然很大,恢复数据的速度比RDB慢。

redis提供了新的持久化方式。将 rdb 文件的内容和增量的 AOF 日志文件存在一起。这里的 AOF 日志不再是全量的日志，而是自持久化开始到持久化结束的这段时间发生的增量 AOF 日志，通常这部分 AOF 日志很小。 

![](../../.gitbook/assets/image%20%2825%29.png)

于是在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志就可以完全替代之前的 AOF 全量文件重放，重启效率因此大幅得到提升。 

