# 前言

本来写完博客就想开启整个复习计划，计划是一天一条博客的。但是前几天因为个人失误我写博客的后台被我删没了……我现在只能直接操控数据库（实在是懒得再写一遍了）。然后刚好又有实训，就一直拖了很久。最近是准备正式开始了。目标计划是Redis -> RabbitMq -> mysql -> springboot -> jvm -> java 有时间可能再把ES写了。希望能自己监督自己把！！

顺便提一嘴，整个博客重构估计是遥遥无期了。感觉是真的好忙。



# Redis入门

## 什么是Redis

Redis（Remote Dictionary Server）即远程字典服务

是一个开源的、支持网络的、可基于内存并且持久化的日志型、key-value型非关系型数据库。其中基于内存要比基于硬盘快1000多倍，但内存无法存储数据，所以需要持久化。

**Redis的作用**

​	1.内存存储，持久化（RDB和AOF）效率高，可以用于高速缓存

​	2.发布订阅系统，队列

​	3.地图信息分析 （geospatial）

​	4.计数器，计时器（比如浏览量 String）

**Redis的特征**

1. 多样的数据结构，五大基本结构和三大特殊结构
2. 持久化
3. 集群
4. 事务



## Redis数据类型

**五个基本类型**

Redis中五大基本类型是：String、List、Hash、Set、Zset。基本操作就不演示了。其中特别说两个

setex(set with expire)：设置过期时间

原子性的操作，要么两个都成功，要么两个都失败

setnx(set if not exist)：不存在设置（在分布式锁中常用）

**三个特殊类型**

Redis中的三大特殊类型是： geospatial、hyperloglog、bitmap

geospatial：redis在3.2推出Geo类型，该功能可以推算出地理位置信息，两地之间的距离

hyperloglog： 基数：数学上集合的元素个数，是不能重复的。这个数据结构通常用于统计网站的UV。

```
127.0.0.1:6379> pfadd mykey a b c d e f g h i j #创建第一组元素
(integer) 1
127.0.0.1:6379> pfcount mykey #统计 mykey的基数
(integer) 10
127.0.0.1:6379> pfadd mykey2 i j z x c v b n m #创建第二组元素
(integer) 1
127.0.0.1:6379> pfcount mykey2 #统计 mykey2 基数
(integer) 9
127.0.0.1:6379> pfmerge mykey3 mykey mykey2 #合并两组mykey mykey2 => mykey3
OK
127.0.0.1:6379> pfcount mykey3
(integer) 15
```

bitmap：就是通过最小的单位bit来进行0或1的设置，表示某个元素对应的值或者状态。一个bit的值，或者是0，或者是1；也就是说一个bit最多能存储的信息是2



## Redis事务

事务的特性：ACID：A(原子性) C(一致性) I(隔离性) D(持久性)。

但Redis的事务不一样。Redis的事务本质上是一组命令的集合。一个事务中的所有命令都会被序列化，执行时会按照顺序执行一次性、顺序性、排他性的执行一条命令。

在Redis中没有事务隔离的概念。因为Redis是单线程。所有的命令在事务中，并没有直接被执行，只有发起执行命令的时候才会执行（exec）。另外Redis单条命令是保证原子性的，但是事务不保证原子性。这是由于redis的本质：redis本质上是命令的集合，Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。

> 正常开启事务

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> mset k3 v3 k4 v4 k5 v5
QUEUED
127.0.0.1:6379> exec
1) OK
2) OK
3) OK
127.0.0.1:6379> keys *
1) "k4"
2) "k2"
3) "k5"
4) "k3"
5) "k1"
127.0.0.1:6379> get k1
"v1"
```

> 放弃事务

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> get k1
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> set k6 v6
QUEUED
127.0.0.1:6379> set k7 v7
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> get k6
(nil)
127.0.0.1:6379> keys *
1) "k4"
2) "k2"
3) "k5"
4) "k3"
5) "k1"
```

> 编译出错 事务中的所有命令不会执行

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k6 v6
QUEUED
127.0.0.1:6379> set k7 v7
QUEUED
127.0.0.1:6379> setget k8 k8
(error) ERR unknown command 'setget'
127.0.0.1:6379> getset k1 aaa
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get k6
(nil)
```

> 运行时异常：如果事务中某条命令执行结果报错，其他命令可以正常执行，错误命令抛出异常

```
127.0.0.1:6379> set t1 v1
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> set k3 3
QUEUED
127.0.0.1:6379> incr k3
QUEUED
127.0.0.1:6379> incr k1
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> get k1
QUEUED
127.0.0.1:6379> get k3
QUEUED
127.0.0.1:6379> exec
1) OK
2) OK
3) (integer) 4
4) (error) ERR value is not an integer or out of range
5) "v2"
6) "v1"
7) "4"
```



### watch

watch命令可以用来监视一个或多个key指令。一旦其中有一个key被修改或者是被删除，那么之后的事务就不能被执行了。监视一致持续到exec命令。（事务中的命令是在exec之后才执行的，所以在multi命令后可以修改watch监控的键值）假设我们通过watch命令在事务执行之前监控了多个keys，倘若在watch之后有任何key的值发生了变化，exec命令执行的事务都将被放弃，同时返回NUull multi-bulk应答已通知调用者事务执行失败。

> Redis 监视测试

正常测试：

```lua
127.0.0.1:6379> set money 100
OK
127.0.0.1:6379> set out 0
OK
127.0.0.1:6379> watch money #监视money对象
OK
127.0.0.1:6379> multi #事务正常结束，执行期间，money没有变动，这个时候就是执行成功了
OK
127.0.0.1:6379> decrby money 20
QUEUED
127.0.0.1:6379> incrby out 20
QUEUED
127.0.0.1:6379> exec
1) (integer) 80
2) (integer) 20
127.0.0.1:6379>
```

测试多线程修改值，使用watch可以当作Redis的乐观锁操作

``` lua
127.0.0.1:6379> set money 100
OK
127.0.0.1:6379> set out 20
OK
127.0.0.1:6379> watch money
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby money 50
QUEUED
127.0.0.1:6379> incrby out 50
QUEUED
127.0.0.1:6379> exec #在执行之前，在另外一个线程B中修改了money的值，下面就是执行失败
(nil)
127.0.0.1:6379> get money
"500"
127.0.0.1:6379>
```

B线程：

```lua
127.0.0.1:6379> set money 500
OK
127.0.0.1:6379>
```

如果修改失败，获取最新的值即可

```lua
127.0.0.1:6379> unwatch #事务执行失败，先解锁
OK
127.0.0.1:6379> watch money #获取最新的值，再次监视 想当于mysql中的select version
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby money 1
QUEUED
127.0.0.1:6379> incrby out 1
QUEUED
127.0.0.1:6379> exec #执行的时候会对比监视的值，如果发生变化执行失败
1) (integer) 499
2) (integer) 21
127.0.0.1:6379>
```



## Redis持久化

redis持久化由两种方式：RDB和AOF。其中RDB默认开启，而AOF默认是不开启的，要去config文件中修改。其中AOF有三种存储方式：appendfsync always（每次操作都持久化）、appendfsync everysec（每一秒持久化）和appendfsync no（不执行持久化）。

## RDB

在指定的时间间隔内，将内存中的数据集快照写入磁盘当中，恢复数据时直接从磁盘中读取，将快照数据恢复到内存中。

**过程**

Redis在RDB持久化过程中，会单独fork出一个子线程去处理。先将数据写进一个临时文件中，待持久化过程结束后，再用这个临时文件去替换上次持久化好的文件。整个过程中主线程不会出现任何IO，这就确保了极高的性能。如果需要大规模的数据恢复，且对数据恢复的完整性不是很敏感，那RDB方式要比AOF更加高效。但是容易出现数据丢失的情况。最后一次持久化的数据可能会丢失。

## AOF

AOF（Append Only File）持久化以独立日志的方式记录每次写命令，并在Redis重启时再重新执行AOF文件中的命令以达到恢复数据的目的，AOF的主要作用是解决数据持久化的实时性。

以日志形式来记录每一个操作，将Redis执行的过程的所有指令记录下来（读操作不记录），只追加文件但不改写文件，redis启动之初会读取该文件重新构建数据结构，换而言之，redis重启的话就根据日志文件的内容将写指令从前到后执行一遍已完成数据的恢复工作。

AOF保存的是appendonly.aof文件



## 各自优缺点

RDB优点：1.RDB文件是某个时间节点的快照，默认使用LZF算法进行压缩，压缩后的文件体积远远小于内存大小，适用于备份、全量复制等场景；2.Redis加载RDB文件恢复数据要远远快于AOF方式
缺点：1.RDB方式实时性不够，无法做到秒级的持久化；2.每次调用bgsave都需要fork子进程，fork子进程属于重量级操作，频繁执行成本较高；3.RDB文件是二进制的，没有可读性，AOF文件在了解其结构的情况下可以手动修改或者补全；4.版本兼容RDB文件问题；
	
AOF优点：1.使用 AOF 持久化会让 Redis 变得非常耐久（much more durable）：你可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync 。 AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求）。
缺点：相对于数据文件来说，AOF远远大于RDB，修复的速度也比RDB慢。2。AOF的运行效率比RDB慢，所以Redis默认的配置就是RDB持久化。

## 拓展

1、RDB持久化方式能够在指定的时间间隔内对你的数据进行快照处理

2、AOF持久化方式记录每次对服务器的写操作，当服务器重启时会重新执行这些命令来恢复原始的数据，AOF命令以Redis协议追加保存每次写的操作到文件尾部，Redis还能对AOF文件进行后台重写，使得AOF文件的体积不至于过大（多次修改同一个值，只保留最后一次修改的命令）

3、只做缓存，如果你只希望你的数据再服务器运行的时候存在，你可以不使用任何持久化操作

4.同时开启两种持久化方式

- 在这种情况下，当Redis重启的时候会优先加载AOF文件来恢复原始的数据，因为在通常情况下，AOF文件保存的数据集要比RDB文件保存的数据集要完整。
- RDB的数据不实时，同步使用两者时服务器重启也只会找AOF，但建议不只使用AOF，因为RDB更适用于备份数据库（AOF在不断的变化不好备份），快速重启，而且不会用AOF可能潜在的BUG，留着作为一个万一的手段。

5、性能建议

- 因为RDB文件只用作后被用途，建议只在Slave上持久化RDB文件，而且只要15分钟备份一次就够了，只保留save 900 1这条规则。
- 如果Enable AOF，好处是在最恶劣的情况下也只会丢失不超过两秒的数据，启动脚本比较简单只load自己的AOF文件就可以了，代价一是带来了持续的IO，二是AOF rewrite 的最后将rewrite过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的。只要硬盘许可，应该尽量减少AOF rewrite 的频率，AOF重写的基础大小默认是64mb太小了，可以设置到5G以上，默认值超过原大小100%时重写。
- 如果不Enable AOF，仅靠 Master-slave Reollcation 实现高可用也可以，能省掉一大笔IO，也较少了rewrite时带来的系统波动，带教师如果Master/slave同时宕掉，会丢失十几分钟的数据，启动脚本也要比较两个Master/Slave中的RDB文件，载入较新的那个，微博就是这种架构