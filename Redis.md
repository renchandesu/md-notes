# Redis

简单来说 Redis 就是一个使用 C 语言开发的内存数据库，所以读写速度非常快，因此 Redis 被广泛应用于缓存方向。

另外，Redis 除了做缓存之外，也经常用来做分布式锁，甚至是消息队列。

Redis 提供了多种数据类型来支持不同的业务场景。Redis 还支持事务 、持久化、Lua 脚本、多种集群方案。


### 基本数据类型
- String 
  - `redis 127.0.0.1:6379> SET name "redis.net.cn"`
  - `redis 127.0.0.1:6379> GET name`
- Hash
  - Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
  - `HMSET user:1 username redis.net.cn password redis.net.cn points 200`
  - `HGETALL user:1`
- List
  - Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。
  - lpush redis.net.cn redis
  - lrange redis.net.cn 0 10
- Set
  - 集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。
  - `sadd key member` 添加一个string元素到,key对应的set集合中，成功返回1,如果元素以及在集合中返回0,key对应的set不存在返回错误。
  - `smembers redis.net.cn`
- zset
  - 每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。
  - 分数可以重复
  - `zadd key score member`
  - `ZRANGEBYSCORE redis.net.cn 0 1000`
- bitmap
  - 一种位操作
  - 本身就是字符串
  - setbit key offset val
  - getbit key offset
  - bitcount key [index1,index2] 统计(字节范围内)1的个数
  - bitop and key key1 key2 ...  
- HyperLogLog
### 基本命令
``redis-cli`` 该命令会连接本地的 redis 服务

``redis-cli -h host -p port -a password``  在远程 redis 服务上执行命令


https://www.redis.net.cn/tutorial/3507.html


### 发布订阅模型
- Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。
- Redis 客户端可以订阅任意数量的频道。
![](../../../../AppData/Local/Temp/pubsub2.png)


### 事务
>Redis 事务可以一次执行多个命令， 并且带有以下两个重要的保证：
> 
>事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
>一个事务从开始到执行会经历以下三个阶段：
>开始事务。
> 
>命令入队。 
> 
>组队
> 
>执行事务。 执行
- 组队时出错任何命令都不执行
- 执行期间出错 只有错误命令不执行 其他都执行
- 事务冲突 
  - 乐观锁 版本控制
    - watch
  - 悲观锁 加锁

### 数据备份与恢复
- Redis SAVE 命令用于创建当前数据库的备份。 手动触发rdb
- 如果需要恢复数据，只需将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可。获取 redis 目录可以使用 CONFIG 命令
- 创建 redis 备份文件也可以使用命令 BGSAVE，该命令在后台执行。 fork一个子线程进行rdb

- RDB 将当前数据生成快照，保存到硬盘 
  - 可配置一定时间执行一次
  - 主从复制时自动执行rdb
  - 关机时，未开启aof会自动rdb
- AOF 通过日志，对数据的修改与写入操作进行记录 这种持久化的方式实时性更好 占用体积大于RDB
  - always
  - ererysec
  - no
- AOF重写 用来缩小aof文件


### 缓存穿透、缓存击穿、缓存雪崩
缓存穿透： 缓存与数据库中根本不存在请求的数据，攻击者不断发送这种请求，使数据库压力过大

缓存击穿： 缓存中不存在，数据库中存在，但是由于并发用户非常多，都去数据库中取数据，导致数据库压力过大

缓存雪崩： 缓存中一大批热点数据过期，查询全部走数据库



### 安全
- `CONFIG set requirepass "w3cschool.cc"`
- `auth "pwd"`


###  Redis 和 Memcached 的区别和共同点

**共同点** ：
1. 都是基于内存的数据库，一般都用来当做缓存使用。
2. 都有过期策略。
3. 两者的性能都非常高。

**区别** ：
1. **Redis 支持更丰富的数据类型（支持更复杂的应用场景）**。Redis 不仅仅支持简单的 k/v 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。Memcached 只支持最简单的 k/v 数据类型。
2. **Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用,而 Memcached 把数据全部存在内存之中。**
3. **Redis 有灾难恢复机制。** 因为可以把缓存中的数据持久化到磁盘上。
4. **Redis 在服务器内存使用完之后，可以将不用的数据放到磁盘上。但是，Memcached 在服务器内存使用完之后，就会直接报异常。**
5. **Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；但是 Redis 目前是原生支持 cluster 模式的。**
6. **Memcached 是多线程，非阻塞 IO 复用的网络模型；Redis 使用单线程的多路 IO 复用模型。** （Redis 6.0 引入了多线程 IO ）
7. **Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持。并且，Redis 支持更多的编程语言。**
8. **Memcached 过期数据的删除策略只用了惰性删除，而 Redis 同时使用了惰性删除与定期删除。**

### Redis 单线程模型详解

**Redis 基于 Reactor 模式来设计开发了自己的一套高效的事件处理模型** （Netty 的线程模型也基于 Reactor 模式，Reactor 模式不愧是高性能 IO 的基石），这套事件处理模型对应的是 Redis 中的文件事件处理器（file event handler）。由于文件事件处理器（file event handler）是单线程方式运行的，所以我们一般都说 Redis 是单线程模型。

**既然是单线程，那怎么监听大量的客户端连接呢？**

Redis 通过**IO 多路复用程序** 来监听来自客户端的大量连接（或者说是监听多个 socket），它会将感兴趣的事件及类型（读、写）注册到内核中并监听每个  事件是否发生。

这样的好处非常明显： **I/O 多路复用技术的使用让 Redis 不需要额外创建多余的线程来监听客户端的大量连接，降低了资源的消耗**（和 NIO 中的 `Selector` 组件很像）。

另外， Redis 服务器是一个事件驱动程序，服务器需要处理两类事件：1. 文件事件; 2. 时间事件。

时间事件不需要多花时间了解，我们接触最多的还是 **文件事件**（客户端进行读取写入等操作，涉及一系列网络通信）。

《Redis 设计与实现》有一段话是如是介绍文件事件的，我觉得写得挺不错。

> Redis 基于 Reactor 模式开发了自己的网络事件处理器：这个处理器被称为文件事件处理器（file event handler）。文件事件处理器使用 I/O 多路复用（multiplexing）程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
>
> 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关 闭（close）等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。
>
> **虽然文件事件处理器以单线程方式运行，但通过使用 I/O 多路复用程序来监听多个套接字**，文件事件处理器既实现了高性能的网络通信模型，又可以很好地与 Redis 服务器中其他同样以单线程方式运行的模块进行对接，这保持了 Redis 内部单线程设计的简单性。

可以看出，文件事件处理器（file event handler）主要是包含 4 个部分：

- 多个 socket（客户端连接）
- IO 多路复用程序（支持多个客户端连接的关键）
- 文件事件分派器（将 socket 关联到相应的事件处理器）
- 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

[![img](https://github.com/Snailclimb/JavaGuide/raw/main/docs/database/redis/images/redis-all/redis%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86%E5%99%A8.png)](https://github.com/Snailclimb/JavaGuide/blob/main/docs/database/redis/images/redis-all/redis事件处理器.png)

《Redis设计与实现：12章》

### Redis 没有使用多线程？为什么不使用多线程？

虽然说 Redis 是单线程模型，但是，实际上，**Redis 在 4.0 之后的版本中就已经加入了对多线程的支持。**

[![redis4.0 more thread](https://github.com/Snailclimb/JavaGuide/raw/main/docs/database/redis/images/redis-all/redis4.0-more-thread.png)](https://github.com/Snailclimb/JavaGuide/blob/main/docs/database/redis/images/redis-all/redis4.0-more-thread.png)

不过，Redis 4.0 增加的多线程主要是针对一些大键值对的删除操作的命令，使用这些命令就会使用主处理之外的其他线程来“异步处理”。

大体上来说，**Redis 6.0 之前主要还是单线程处理。**

**那，Redis6.0 之前 为什么不使用多线程？**

我觉得主要原因有下面 3 个：

1. 单线程编程容易并且更容易维护；
2. Redis 的性能瓶颈不在 CPU ，主要在内存和网络；
3. 多线程就会存在死锁、线程上下文切换等问题，甚至会影响性能。

### Redis6.0 之后为何引入了多线程？

**Redis6.0 引入多线程主要是为了提高网络 IO 读写性能**，因为这个算是 Redis 中的一个性能瓶颈（Redis 的瓶颈主要受限于内存和网络）。

虽然，Redis6.0 引入了多线程，但是 Redis 的多线程只是在网络数据的读写这类耗时操作上使用了，执行命令仍然是单线程顺序执行。因此，你也不需要担心线程安全问题。



### Redis 为什么需要用到锁

因为虽然redis内部数据的处理是one by one 的，但是取出数据后，语言对数据进行操作后再存入，其实是并行的，所以需要加锁（乐观锁 悲观锁）