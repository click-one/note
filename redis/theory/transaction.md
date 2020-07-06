# 事务

## redis事务的基本使用

每个事务的操作都有 begin、commit 和 rollback，在java中的使用形式

```java
begin();
try {
	command1();
	command2();
	...
	commit();
} catch(Exception e) {
	rollback();
}
```

Redis 在形式上看起来也差不多，分别是 multi/exec/discard。multi 指示事务的开始，exec 指示事务的执行，discard 指示事务的丢弃。 

```java
127.0.0.1:6379> multi   #开启事务
OK
127.0.0.1:6379> incr books  #执行命令
QUEUED
127.0.0.1:6379> exec        #提交事务
1) (integer) 1
```

所有的指令在 exec 之前不执行，而是缓存在服务器的一个事务队列中，服务器一旦收到 exec 指令，才开执行整个事务队列，执行完毕后一次性返回所有指令的运行结果。因为 Redis 的单线程特性，它不用担心自己在执行队列的时候被其它指令打搅。

## 原子性

```java
127.0.0.1:6379> multi
OK
127.0.0.1:6379> incr books
QUEUED
127.0.0.1:6379> set aaa bbb
QUEUED
127.0.0.1:6379> incr aaa
QUEUED
127.0.0.1:6379> incr aaa
QUEUED
127.0.0.1:6379> incr books
QUEUED
127.0.0.1:6379> exec
1) (integer) 2
2) OK
3) (error) ERR value is not an integer or out of range
4) (error) ERR value is not an integer or out of range
5) (integer) 3
```

从上面可以看出，事务里面中间过程出错了，后面的命令依旧在执行，redis中的事务不能满足原子性。

## watch命令

Redis 提供了 watch 机制，它就是一种乐观锁。有了 watch 我们又多了一种可以用来解决并发修改的方法。 

```java
127.0.0.1:6379> watch books
OK
127.0.0.1:6379> incr books
(integer) 6
127.0.0.1:6379> multi
OK
127.0.0.1:6379> incr books
QUEUED
127.0.0.1:6379> exec
(nil)
```

watch 会在事务开始之前盯住 1 个或多个关键变量，当事务执行时，也就是服务器收到了 exec 指令要顺序执行缓存的事务队列时，Redis 会检查关键变量自 watch 之后，是否被修改了 \(包括当前事务所在的客户端\)。如果关键变量被人动过了，exec 指令就会返回 null 回复告知客户端事务执行失败。

