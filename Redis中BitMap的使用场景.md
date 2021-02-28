# [Redis 中 BitMap 的使用场景](https://www.cnblogs.com/54chensongxia/p/13794391.html)



## BitMap[#](https://www.cnblogs.com/54chensongxia/p/13794391.html#802193944)

BitMap 原本的含义是用一个比特位来映射某个元素的状态。由于一个比特位只能表示 0 和 1 两种状态，所以 BitMap 能映射的状态有限，但是使用比特位的优势是能大量的节省内存空间。

在 Redis 中，可以把 Bitmaps 想象成一个以比特位为单位的数组，数组的每个单元只能存储0和1，数组的下标在 Bitmaps 中叫做偏移量。

[![img](https://easyreadfs.nosdn.127.net/bFxn73tfrlnNzJDOe-0WzA==/8796093023252283210)](https://easyreadfs.nosdn.127.net/bFxn73tfrlnNzJDOe-0WzA==/8796093023252283210)

需要注意的是：BitMap 在 Redis 中并不是一个新的数据类型，其底层是 Redis 实现。

## BitMap 相关命令[#](https://www.cnblogs.com/54chensongxia/p/13794391.html#2026054576)

```
Copy# 设置值，其中value只能是 0 和 1
setbit key offset value

# 获取值
getbit key offset

# 获取指定范围内值为 1 的个数
# start 和 end 以字节为单位
bitcount key start end

# BitMap间的运算
# operations 位移操作符，枚举值
  AND 与运算 &
  OR 或运算 |
  XOR 异或 ^
  NOT 取反 ~
# result 计算的结果，会存储在该key中
# key1 … keyn 参与运算的key，可以有多个，空格分割，not运算只能一个key
# 当 BITOP 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 0。返回值是保存到 destkey 的字符串的长度（以字节byte为单位），和输入 key 中最长的字符串长度相等。
bitop [operations] [result] [key1] [keyn…]

# 返回指定key中第一次出现指定value(0/1)的位置
bitpos [key] [value]
```

## BitMap 占用的空间[#](https://www.cnblogs.com/54chensongxia/p/13794391.html#1621910516)

在弄清 BitMap 到底占用多大的空间之前，我们再来重申下：**Redis 其实只支持 5 种数据类型，并没有 BitMap 这种类型，BitMap 底层是基于 Redis 的字符串类型实现的。**

我们通过下面的命令来看下 BitMap 占用的空间大小：

```
Copy# 首先将偏移量是0的位置设为1
127.0.0.1:6379> setbit csx:key:1 0 1
(integer) 0
# 通过STRLEN命令，我们可以看到字符串的长度是1
127.0.0.1:6379> STRLEN csx:key:1
(integer) 1
# 将偏移量是1的位置设置为1
127.0.0.1:6379> setbit csx:key:1 1 1
(integer) 0
# 此时字符串的长度还是为1，以为一个字符串有8个比特位，不需要再开辟新的内存空间
127.0.0.1:6379> STRLEN csx:key:1
(integer) 1
# 将偏移量是8的位置设置成1
127.0.0.1:6379> setbit csx:key:1 8 1
(integer) 0
# 此时字符串的长度编程2，因为一个字节存不下9个比特位，需要再开辟一个字节的空间
127.0.0.1:6379> STRLEN csx:key:1
(integer) 2
```

通过上面的实验我们可以看出，BitMap 占用的空间，就是底层字符串占用的空间。假如 BitMap 偏移量的最大值是 OFFSET_MAX，那么它底层占用的空间就是：

```
Copy(OFFSET_MAX/8)+1 = 占用字节数
```

因为字符串内存只能以字节分配，所以上面的单位是字节。

但是需要注意，Redis 中字符串的最大长度是 512M，所以 BitMap 的 offset 值也是有上限的，其最大值是：

```
Copy8 * 1024 * 1024 * 512  =  2^32
```

由于 C语言中字符串的末尾都要存储一位分隔符，所以实际上 BitMap 的 offset 值上限是：

```
Copy(8 * 1024 * 1024 * 512) -1  =  2^32 - 1
```

## 使用场景[#](https://www.cnblogs.com/54chensongxia/p/13794391.html#943459957)

**1. 用户签到**

很多网站都提供了签到功能，并且需要展示最近一个月的签到情况，这种情况可以使用 BitMap 来实现。
根据日期 offset = （今天是一年中的第几天） % （今年的天数），key = 年份：用户id。

如果需要将用户的详细签到信息入库的话，可以考虑使用一个异步线程来完成。

**2. 统计活跃用户（用户登陆情况）**

使用日期作为 key，然后用户 id 为 offset，如果当日活跃过就设置为1。具体怎么样才算活跃这个标准大家可以自己指定。

假如 20201009 活跃用户情况是： [1，0，1，1，0]
20201010 活跃用户情况是 ：[ 1，1，0，1，0 ]

统计连续两天活跃的用户总数：

```
Copybitop and dest1 20201009 20201010 
# dest1 中值为1的offset，就是连续两天活跃用户的ID
bitcount dest1
```

统计20201009 ~ 20201010 活跃过的用户：

```
Copybitop or dest2 20201009 20201010 
```

**3. 统计用户是否在线**

如果需要提供一个查询当前用户是否在线的接口，也可以考虑使用 BitMap 。即节约空间效率又高，只需要一个 key，然后用户 id 为 offset，如果在线就设置为 1，不在线就设置为 0。

**4. 实现布隆过滤器**

# HyperLogLog

算法实现https://www.jianshu.com/p/55defda6dcd2

基数就是指一个集合中不同值的数目，比如[a,b,c,d]的基数就是4，[a,b,c,d,a]的基数还是4，因为a重复了一个，不算。基数也可以称之为Distinct Value，简称DV。HyperLogLog算法经常在数据库中被用来统计某一字段的Distinct Value，HyperLogLog算法就是用来计算基数的。

Redis的所有HyperLogLog结构都是固定的16384个桶（2的14次方），并且有两种存储格式：

- 稀疏格式：HyperLogLog算法在刚开始的时候，大多数桶其实都是0，稀疏格式通过存储连续的0的数目，而不是每个0存一遍，大大减小了HyperLogLog刚开始时需要占用的内存
- 紧凑格式：用6个bit表示一个桶，需要占用12KB内存

**应用场景**

　　基数不大，数据量不大就用不上，会有点大材小用浪费空间，有局限性，就是只能统计基数数量，而没办法去知道具体的内容是什么，和bitmap相比，属于两种特定统计情况，简单来说，HyperLogLog 去重比 bitmap 方便很多，一般可以bitmap和hyperloglog配合使用，bitmap标识哪些用户活跃，hyperloglog计数
　　一般使用：

　　　　统计注册 IP 数
　　　　统计每日访问 IP 数
　　　　统计页面实时 UV 数
　　　　统计在线用户数
　　　　统计用户每天搜索不同词条的个数

# Redis GEO

Redis GEO 主要用于存储地理位置信息，并对存储的信息进行操作，该功能在 Redis 3.2 版本新增。

Redis GEO 操作方法有：

- geoadd：添加地理位置的坐标。
- geopos：获取地理位置的坐标。
- geodist：计算两个位置之间的距离。
- georadius：根据用户给定的经纬度坐标来获取指定范围内的地理位置集合。
- georadiusbymember：根据储存在位置集合里面的某个地点获取指定范围内的地理位置集合。
- geohash：返回一个或多个位置对象的 geohash 值。

# 关于Redis Module

像BloomFilter，RedisSearch，Redis-ML





假如Redis⾥⾯有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如何将它们
全部找出来？
	使⽤keys指令可以扫出指定模式的key列表。
如果这个redis正在给线上的业务提供服务，那使⽤keys指令会有什么问题？
	这个时候你要回答redis关键的⼀个特性：redis的单线程的。keys指令会导致线程阻塞⼀段时间，线上
服务会停顿，直到指令执⾏完毕，服务才能恢复。这个时候可以使⽤scan指令，scan指令可以⽆阻塞
的提取出指定模式的key列表，但是会有⼀定的重复概率，在客户端做⼀次去重就可以了，但是整体所
花费的时间会⽐直接⽤keys指令⻓。
	不过，增量式迭代命令也不是没有缺点的： 举个例⼦， 使⽤ SMEMBERS 命令可以返回集合键当前包
含的所有元素， 但是对于 SCAN 这类增量式迭代命令来说， 因为在对键进⾏增量式迭代的过程中， 键
可能会被修改， 所以增量式迭代命令只能对被返回的元素提供有限的保证 。