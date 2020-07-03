# 通信协议

## **RESP\(Redis Serialization Protocol\)** 

Redis 协议将传输的结构数据分为 5 种最小单元类型，单元结束时统一加上回车换行符号\r\n。 

* 单行字符串 以 + 符号开头。 
* 多行字符串 以 $ 符号开头，后跟字符串长度。 
* 整数值 以 : 符号开头，后跟整数的字符串形式。 
* 错误消息 以 - 符号开头。 
* 数组 以 \* 号开头，后跟数组的长度。 

单行字符串 `hello world +hello world\r\n` 

多行字符串 `hello world $11\r\nhello world\r\n` 多行字符串当然也可以表示单行字符串。

 整数 `1024 :1024\r\n` 

错误 参数类型错误 `-WRONGTYPE Operation against a key holding the wrong kind of value` 

数组 `[1,2,3] *3\r\n:1\r\n:2\r\n:3\r\n`

 NULL 用多行字符串表示，不过长度要写成-1。 `$-1\r\n` 

空串 用多行字符串表示，长度填 0。 `$0\r\n\r\n`

## 客户端请求服务端

客户端向服务器发送的指令只有一种格式，多行字符串数组。比如一个简单的 set 指令set author codehole会被序列化成下面的字符串。 `*3\r\n$3\r\nset\r\n$6\r\nauthor\r\n$8\r\ncodehole\r\n`

## 服务端响应客户端

### 单行字符串响应 

```text
127.0.0.1:6379> set author lvlei
OK
```

###  错误响应

```text
127.0.0.1:6379> incr author
(error) ERR value is not an integer or out of range
```

### 整数响应 

```text
127.0.0.1:6379> incr age
(integer) 2
```

### 嵌套 

```text
127.0.0.1:6379> scan 0
1) "0"
2) 1) "codehole"
   2) "company"
   3) "name"
   4) "age"
   5) "author"
```

scan 命令用来扫描服务器包含的所有 key 列表，它是以游标的形式获取，一次只获取一部分。 scan 命令返回的是一个嵌套数组。

数组的第一个值表示游标的值，如果这个值为零，说明已经遍历完毕。如果不为零，使用这个值作为 scan 命令的参数进行下一次遍历。数组的第二个值又是一个数组，这个数组就是 key 列表。

