# 小对象压缩

## 小对象压缩存储 **\(ziplist\)** 

如果 Redis 内部管理的集合数据结构很小，它会使用紧凑存储形式压缩存储。 

Redis 的 ziplist 是一个紧凑的字节数组结构，如下图所示，每个元素之间都是紧挨着的

![](../../.gitbook/assets/image%20%2830%29.png)

**如果它存储的是 hash 结构，那么 key 和 value 会作为两个 entry 相邻存在一起。** 

```text
127.0.0.1:6379> hset student a1 xiaoming
(integer) 1
127.0.0.1:6379> hset student a2 xiaoli
(integer) 1
127.0.0.1:6379> hset student a3 xiaowang
(integer) 1
127.0.0.1:6379> object encoding student
"ziplist"
```

**如果它存储的是** _**zset**_**，那么 value 和 score 会作为两个 entry 相邻存在一起。** 

```text
127.0.0.1:6379> zadd world 1 a
(integer) 1
127.0.0.1:6379> zadd world 2 b
(integer) 1
127.0.0.1:6379> zadd world 3 c
(integer) 1
127.0.0.1:6379> object encoding world
"ziplist"
```

**intset 是一个紧凑的整数数组结构，它用于存放元素都是整数的并且元素个数较少的 set 集合。**

```text
127.0.0.1:6379> sadd hello 1 2 3
(integer) 3
127.0.0.1:6379> object encoding hello
"intset"
127.0.0.1:6379> sadd hello yes no
(integer) 2
127.0.0.1:6379> object encoding hello
"hashtable"
#如果 set 里存储的是字符串，那么 sadd 立即升级为 hashtable 结构。
```

存储界限 当集合对象的元素不断增加，或者某个 value 值过大，这种小对象存储也会被升级为标准结构。Redis 规定在小对象存储结构的限制条件如下：

* hash-max-zipmap-entries 512 \# hash 的元素个数超过 512 就必须用标准结构存储 
* hash-max-zipmap-value 64 \# hash 的任意元素的 key/value 的长度超过 64 就必须用标准结构存储 
* list-max-ziplist-entries 512 \# list 的元素个数超过 512 就必须用标准结构存储
*  list-max-ziplist-value 64 \# list 的任意元素的长度超过 64 就必须用标准结构存储 
* zset-max-ziplist-entries 128 \# zset 的元素个数超过 128 就必须用标准结构存储 
* zset-max-ziplist-value 64 \# zset 的任意元素的长度超过 64 就必须用标准结构存储 
* set-max-intset-entries 512 \# set 的整数元素个数超过 512 就必须用标准结构存储

