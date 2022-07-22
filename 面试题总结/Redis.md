## Redis

### 1. Redis可以用来做什么？

==参考答案==

1. Redis最常用来做缓存，是实现分布式缓存的首先中间件；
2. Redis可以作为数据库，实现诸如点赞、关注、排行等对性能要求极高的互联网需求；
3. Redis可以作为计算工具，能用很小的代价，通过 HyperLogLog 统计 PV/UV、通过 BitMap 统计用户在线天数、签到次数等数据；
4. Redis还有很多其他的使用场景，例如：可以实现分布式锁，可以基于 stream 作为消息队列使用。
5. Redis还可以做注册中心（Redis 哨兵模式和集群模式）

> **UV**：全称 **Unique Visitor**，也叫 **独立访客量**，是指通过互联网访问、浏览这个网页的自然人。1天内同一个用户多次访问该网站，只记录1次。
>
> **PV**：全称 **Page View**，也叫 **页面访问量** 或 **页面点击量**，用户每访问网站的一个页面，记录1次 PV，用户多次打开页面，则记录多次 PV。往往用来衡量网站的流量。

### 2. Redis 和传统的关系型数据库有什么不同？

==参考答案==

​		Redis是一种基于键值对的NoSQL数据库，而键值对的值是由多种数据结构和算法组成的。Redis的数据都存储于内存中，因此它的速度惊人，读写性能可达10万/秒，远超关系型数据库。

​		关系型数据库是基于二维数据表来存储数据的，它的数据格式更为严谨，并支持关系查询。关系型数据库的数据存储于磁盘上，可以存放海量的数据，但性能远不如Redis。

### 3. Redis有哪些数据类型？

==参考答案==

1. Redis支持5种核心的数据类型，分别是字符串（String）、哈希（Hash）、列表（List）、集合（Set）、有序集合（SortedSet）；
   - **String** 类型的三种格式：字符串、int、float
   - **Hash** 类型，也叫散列，其 value 是一个无序字典，类似于 Java 中的 HashMap 结构
   - **List** 类型与 Java 中的 LinkedList 类似，可以看做是一个**双向链表结构**
   - **Set** 类型与 Java 中的 HashSet 类似，可以看做是一个 value 为 null 的 HashMap
   - **SortedSet** 类型是一个可排序的 set 集合，与 Java 中的 TreeSet 有些类似，但底层数据结构却差别很大。SortedSet 中的每一个元素都带有一个 score 属性，可以基于 score 属性对元素排序，底层的实现是一个跳表（SkipList）加 hash表。
2. Redis还提供了Bitmap、HyperLogLog、Geo类型，但这些类型都是基于上述核心数据类型实现的；
3. Redis在5.0新增加了 Streams 数据类型，它是一个功能强大的、支持多播的、可持久化的消息队列。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220720233403627.png" alt="image-20220720233403627" style="zoom: 67%;" />

### 4. Redis是单线程的，为什么还能这么快？

==参考答案==

1. 对服务端程序来说，线程切换和锁通常是性能杀手，而单线程避免了线程切换和竞争所产生的消耗；
2. Redis的大部分操作是在内存上完成的，这是它实现高性能的一个重要原因；
3. Redis采用了IO多路复用机制，使其在网络IO操作中能并发处理大量的客户端请求，实现高吞吐率。

关于Redis的单线程架构实现，如下图：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/7D358C4626AF51725C251A2611C5DD65" alt="img" style="zoom: 50%;" />

### 5. Redis在持久化时fork出一个子进程，这时已经有两个进程了，怎么能说是单线程呢？

==参考答案==

Redis是单线程的，主要是指Redis的网络IO和键值对读写是由一个线程来完成的。而Redis的其他功能，如持久化、异步删除、集群数据同步等，则是依赖其他线程来执行的。所以，说Redis是单线程的只是一种习惯的说法，事实上它的底层不是单线程的。

### 6. set和zset有什么区别？ 

==参考答案==

set：

- 集合中的元素是无序、不可重复的，一个集合最多能存储 $2^{32}-1$ 个元素；
- 集合除了支持对元素的增删改查之外，还支持对多个集合取交集、并集、差集。

zset：

- 有序集合保留了集合元素不能重复的特点；
- 有序集合会给每个元素设置一个 score 属性，并以此作为排序的依据；
- 有序集合不能包含相同的元素，但是不同元素的分数可以相同。

### 7. Redis 事务

#### 7.1 如何使用 Redis 事务？

Redis 可以通过 **`MULTI`，`EXEC`，`DISCARD` 和 `WATCH`** 等命令来实现事务(transaction)功能。

```sh
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> set name ayi
QUEUED
127.0.0.1:6379(TX)> get name
QUEUED
127.0.0.1:6379(TX)> setex age 60 20
QUEUED
127.0.0.1:6379(TX)> expire name 60
QUEUED
127.0.0.1:6379(TX)> EXEC
1) OK
2) "ayi"
3) OK
4) (integer) 1
```

-  **[`MULTI`](https://redis.io/commands/multi)** 和 [`EXEC`](https://redis.io/commands/exec) ：使用 `MULTI`命令后可以输入多个命令。Redis 不会立即执行这些命令，而是将它们放到队列，当调用了 `EXEC` 命令将执行所有命令。

这个过程是这样的：

1. 开始事务（`MULTI`）。
2. 命令入队(批量操作 Redis 的命令，先进先出（FIFO）的顺序执行)。
3. 执行事务(`EXEC`)。

- **[`DISCARD`](https://redis.io/commands/discard)** ：你也可以通过 `DISCARD` 命令取消一个事务，它会清空事务队列中保存的所有命令。

```sh
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> set name ayi
QUEUED
127.0.0.1:6379(TX)> DISCARD
OK
127.0.0.1:6379>
```

- **[`WATCH`](https://redis.io/commands/watch)** ：命令用于监听指定的键，当调用 `EXEC` 命令执行事务时，如果一个被 `WATCH` 命令监视的键被修改的话，整个事务都不会执行，直接返回失败。

  很多时候，要确保事务中的数据没有被其他客户端修改才执行该事务。Redis提供了 watch 命令来解决这类问题，这是一种**乐观锁**的机制。客户端通过 watch 命令，要求服务器对一个或多个 key 进行监视，如果在客户端执行事务之前，这些key发生了变化，则服务器将拒绝执行客户端提交的事务，并向它返回一个空值。

#### 7.2  Redis 支持原子性吗？

Redis 的事务和我们平时理解的关系型数据库的事务不同。我们知道事务具有四大特性 ACID： **1. 原子性**，**2. 一致性**，**3. 持久性**，**4. 隔离性**。

**Redis 事务在运行错误的情况下，除了执行过程中出现错误的命令外，其他命令都能正常执行。并且，Redis 是不支持回滚（roll back）操作的。因此，Redis 事务其实是不满足原子性的（而且不满足持久性）。**

可以将 Redis 中的事务就理解为 ：**Redis 事务提供了一种将多个命令请求打包的功能。然后，再按顺序执行打包的所有命令，并且不会被中途打断。**

```sh
redis＞MULTI
OK
redis＞SET key 1
QUEUED
redis＞SADD key 2
QUEUED
redis＞SET key 3
QUEUED
redis＞EXEC
1) OK
2) (error) ERR Operation against a key holding the wrong kind of value
3) OK
redis＞GET key
"3"
```

Redis 官网也解释了自己为啥不支持回滚。简单来说就是 Redis 开发者们觉得没必要支持回滚，这样更简单便捷并且性能更好。Redis 开发者觉得即使命令执行错误也应该在开发过程中就被发现而不是生产过程中。

#### 7.3 如何解决 Redis 事务的缺陷？

Redis 从 2.6 版本开始支持执行 Lua 脚本，它的功能和事务非常类似。我们可以利用 Lua 脚本来批量执行多条 Redis 命令，这些 Redis 命令会被提交到 Redis 服务器一次性执行完成，大幅减小了网络开销。

一段 Lua 脚本可以视作一条命令执行，一段 Lua 脚本执行过程中不会有其他脚本或 Redis 命令同时执行，保证了操作不会被其他指令插入或打扰。

如果 Lua 脚本运行时出错并中途结束，出错之后的命令是不会被执行的。并且，出错之前执行的命令是无法被撤销的。因此，严格来说，通过 Lua 脚本来批量执行 Redis 命令也是不满足原子性的。

### 8. 如何利用Redis实现一个分布式锁？

==参考答案==

#### 8.1何时需要分布式锁？

​		在分布式的环境下，当多个 server 并发修改同一个资源时，为了避免竞争就需要使用分布式锁。那为什么不能使用Java自带的锁呢？因为Java中的锁是面向多线程设计的，它只局限于当前的JRE环境。而多个server实际上是多进程，是不同的JRE环境，所以Java自带的锁机制在这个场景下是无效的。

#### 8.2 如何实现分布式锁？

##### 8.2.1 采用单点Redis实现分布式锁

就是在Redis里存一份代表锁的数据，通常用字符串即可。实现分布式锁的思路，以及优化的过程如下：

1. 加锁：

   第一版，这种方式的缺点是容易产生死锁，因为客户端有可能忘记解锁，或者解锁失败。

   ```
   setnx key value
   ```

   第二版，给锁增加了过期时间，避免出现死锁。但这两个命令不是原子的，第二步可能会失败，依然无法避免死锁问题。

   ```redis
   setnx key value 
   expire key seconds
   ```

   第三版，通过“set...nx...”命令，将加锁、过期命令编排到一起，它们是原子操作了，可以避免死锁。

   ```
   set key value nx ex seconds 
   ```

2. 解锁：

   解锁就是删除代表锁的那份数据。

   ```
   del key
   ```

3. 问题：

   看起来已经很完美了，但实际上还有隐患，如下图。线程1在任务没有执行完毕时发生业务阻塞，锁已经超时被释放了，此后不久线程2获取到了锁。等线程1的任务执行结束后，它依然会尝试释放锁，因为它的代码逻辑就是任务结束后释放锁。但是，它的锁早已自动释放过了，它此时释放的就是线程2的锁。


<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220721231813759.png" alt="image-20220721231813759" style="zoom:50%;" />

想要解决这个问题，我们需要解决两件事情：

1. 在加锁时就要给锁设置一个标识，线程要记住这个标识。当线程释放锁的时候，要进行判断，是自己持有的锁才能释放，否则不能释放。可以为key赋一个随机值（例如UUID + 线程唯一标识threadId），来充当线程的标识。
2. 释放锁时要先判断、再释放，这两步需要保证原子性，否则第二步失败的话，就会出现死锁。而获取和删除命令不是原子的，这就需要采用Lua脚本，通过Lua脚本将两个命令编排在一起，而整个Lua脚本的执行是原子的。

按照以上思路，优化后的命令如下：

```sh
-- 加锁 
set key random-value nx ex seconds
```

```lua
-- 解锁 
if (redis.call('get',KEYS[1]) == ARGV[1]) then    
    return redis.call('del',KEYS[1]) 
else     
  	return 0 
end
```

基于setnx实现的分布式锁存在下面的问题：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220721234432888.png" alt="image-20220721234432888" style="zoom: 50%;" />



> **不可重入：**
>
> 一个线程的多个执行方法只能获取一次锁，不可重入。
>
> <img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722000016648.png" alt="image-20220722000016648" style="zoom: 50%;" />
>
> **Redission 可重入锁的实现原理：**
>
> 不能再使用 String 类型来存储锁了，需要用 Hash 类型来存储，其中 field 存储线程标识，value 存储重入次数（每重入一次，就将对应 key.field 的 key.value值 +1，每释放一次就-1，当value == 0 时，将锁删除），获取锁和释放锁的过程要保证原子性，需要通过 lua 脚本实现。
>
> <img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722002107881.png" alt="image-20220722000016648" style="zoom: 50%;" />
>
> ```lua
> -- 获取可重入锁getReentrantLock
> 
> local key = KEYS[1]; --锁的key
> local threadId = ARGV[1]; --线程的唯一标识
> local releaseTime = ARGV[2]; -- 锁的自动释放时间
> -- 判断是否存在
> if(redis.call('exists', key) == 0) then
>     -- 不存在，获取锁
>     redis.call('hset', key, threadId, '1');
>     -- 设置有效期
>     redis.call('expire', key, releaseTime);
> end
> -- 锁已经存在，判断threadId是否是自己
> if(redis.call('hexists', key, threadId) == 1) then
>     -- 获取锁，重入次数+1
>     redis.call('hincrby', key, threadId, '1');
>     -- 设置有效期
>     redis.call('expire', key, releaseTime);
>     return 1; -- 返回结果
> end
> return 0; --表示获取失败
> ```
>
> ```lua
> -- 释放可重入锁getReentrantLock
> 
> local key = KEYS[1]; --锁的key
> local threadId = ARGV[1]; --线程的唯一标识
> local releaseTime = ARGV[2]; -- 锁的自动释放时间
> 
> -- 判断当前锁是否还是被自己持有
> if(redis.call('hexists', key, threadId) == 0) then
>     return nil; -- 如果已经不是自己，则直接返回
> end
> 
> -- 是自己的锁，释放时，重入次数-1
> local count = redis.call('hincrby', key, threadId, -1);
> if(count > 0) then
>     redis.call('expire', key, releaseTime);
>     return nil;
> else
>     redis.call('del', key)
>     return nil;
> end
> ```
>
> // TODO Redisson实现可重试等原理

##### 8.2.2 采用 Redis 集群实现分布式锁

> 经过刚刚的讨论，我们已经有较好的方法获取锁和释放锁。基于Redis单实例，假设这个单实例总是可用，这种方法已经足够安全。如果这个Redis节点挂掉了呢？

到这个问题其实可以直接聊到 Redlock 了。但是你别慌啊，为了展示你丰富的知识储备，你得先自己聊一聊 Redis 的集群，你可以这样去说：

为了避免节点挂掉导致的问题，我们可以采用Redis集群的方法来实现Redis的高可用。

**Redis集群方式共有三种：主从模式，哨兵模式，cluster(集群)模式**

- **主从模式** 会保证数据在从节点还有一份，但是主节点挂了之后，需要手动把从节点切换为主节点。它非常简单，但是在实际的生产环境中是很少使用的。

- **哨兵模式** 就是主从模式的升级版，该模式下会对响应异常的主节点进行主观下线或者客观下线的操作，并进行主从切换。它可以保证高可用。

- **集群模式** 保证的是高并发，整个集群分担所有数据，不同的 key 会放到不同的 Redis 中。每个 Redis 对应一部分的槽。

> 在上面描述的集群模式下还是会出现一个问题，**由于节点之间是采用异步通信的方式，如果刚刚在 Master 节点上加了锁，但是数据还没被同步到 Salve，这时 Master 节点挂了，它上面的锁就没了，等新的 Master 出来后（主从模式的手动切换或者哨兵模式的一次 failover 的过程），就可以再次获取同样的锁，出现一把锁被拿到了两次的场景。**

锁都被拿了两次了，也就不满足安全性了。**一个安全的锁，不管是不是分布式的，在任意一个时刻，都只有一个客户端持有。**

##### 8.2.3 基于RedLock算法的分布式锁

由于集群模式下分布式锁仍然存在锁的不唯一性的问题，为了解决这个的问题，Redis 的作者提出了名为 **Redlock** 的算法。该算法基于多个 Redis 节点，它的基本逻辑如下：

- 这些节点相互独立，不存在主从复制或者集群协调机制（每个节点都可以视为 Mater 节点）；
- 加锁：以相同的 `KEY` 向 N 个实例加锁，只要超过一半节点成功，则认定加锁成功；
- 解锁：向所有的实例发送 `DEL` 命令，进行解锁；

**RedLock** 算法思想，意思是不能只在一个 redis 实例上创建锁，应该是在多个 redis 实例上创建锁，**n / 2 + 1**，必须在大多数 redis 节点上都成功创建锁，才能算这个整体的 **RedLock** 加锁成功，避免说仅仅在一个 redis 实例上加锁而带来的问题。



RedLock，这里是分别对3个 redis 实例加锁，然后获取一个最后的加锁结果：

```java
RLock lock1 = redisson1.getLock("lock1");
RLock lock2 = redisson2.getLock("lock2");
RLock lock3 = redisson3.getLock("lock3");

RLock redLock = anyRedisson.getRedLock(lock1, lock2, lock3);

// traditional lock method
redLock.lock();

// or acquire lock and automatically unlock it after 10 seconds
redLock.lock(10, TimeUnit.SECONDS);

// or wait for lock aquisition up to 100 seconds 
// and automatically unlock it after 10 seconds
boolean res = redLock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       redLock.unlock();
   }
}
```



**RedLock** 算法的示意图如下，我们可以自己实现该算法，也可以直接使用 Redisson 框架。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/5B9C19EC2AFBB10977147566DF33BE03" alt="img" style="zoom: 50%;" />

只要大多数的节点可以正常工作，就可以保证 Redlock 的正常工作。这样就可以解决前面单点 Redis 的情况下我们讨论的节点挂掉，由于异步通信，导致锁失效的问题。

> 但是，还是不能解决故障重启后带来的锁的安全性的问题。你想一下下面这个场景：
>
> 
>
> 我们一共有 A、B、C 这三个节点。
>
> 1. 客户端 1 在 A，B 上加锁成功。C 上加锁失败。
> 2. 这时节点 B 崩溃重启了，但是由于持久化策略可能导致客户端 1 在 B 上的锁没有持久化下来。
> 3. 客户端 2 发起申请同一把锁的操作，在 B，C 上加锁成功。
> 4. 这个时候就又出现同一把锁，同时被客户端 1 和客户端 2 所持有了。

参考：

1. *Redis锁从面试连环炮聊到神仙打架* *https://mp.weixin.qq.com/s?__biz=Mzg3NjU3NTkwMQ==&mid=2247505097&idx=1&sn=5c03cb769c4458350f4d4a321ad51f5a&source=41#wechat_redirect*

2. *03-使用Redisson实现RedLock原理* https://cloud.tencent.com/developer/article/1602970

---

### 9.Redis的持久化策略

==参考答案==

Redis支持**RDB持久化**、**AOF持久化**和**RDB-AOF混合持久化**这三种持久化方式。

#### 9.1RDB持久化

**RDB(Redis Database)**是Redis默认采用的持久化方式，它以快照（Snapshot）的形式将进程数据持久化到硬盘中。RDB会创建一个经过压缩的二进制文件，文件以`.rdb`结尾，内部存储了各个数据库的键值对数据等信息。RDB持久化的触发方式有两种：

- **手动触发**：通过 SAVE 或 BGSAVE 命令触发 RDB 持久化操作，创建 “.rdb” 文件；
- **自动触发**：通过配置选项，让服务器在满足指定条件时自动执行 BGSAVE 命令。

其中，`SAVE`命令执行期间，Redis服务器将阻塞，直到`.rdb`文件创建完毕为止。而`BGSAVE`命令是异步版本的`SAVE`命令，它会使用Redis服务器进程的子进程，创建`.rdb`文件。`BGSAVE`命令在创建子进程时会存在短暂的阻塞，之后服务器便可以继续处理其他客户端的请求。总之，`BGSAVE`命令是针对`SAVE`阻塞问题做的优化，Redis内部所有涉及RDB的操作都采用`BGSAVE`的方式，而 SAVE 命令已经废弃！



快照持久化是 Redis 默认采用的持久化方式，在 `redis.conf` 配置文件中默认有此下配置：

```properties
# 在900秒(15分钟)之内，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
save 900 1           
# 在300秒(5分钟)之内，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
save 300 10          
# 在60秒(1分钟)之内，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
save 60 10000        
```

BGSAVE命令的执行流程，如下图：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722011308551.png" alt="image-20220722011308551" style="zoom:50%;" />

BGSAVE 命令的原理，如下图：

> fork采用的是copy-on-write技术：
>
> - 当主进程执行读操作时，访问共享内存；
> - 当主进程执行写操作时，则会拷贝一份数据，执行写操作。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722012126098.png" alt="image-20220722012126098" style="zoom:50%;" />



RDB持久化的优缺点如下：

- 优点：RDB生成紧凑压缩的二进制文件，体积小，使用该文件恢复数据的速度非常快；
- 缺点：BGSAVE每次运行都要执行 fork 操作创建子进程，属于重量级操作，不宜频繁执行，所以 RDB 持久化没办法做到实时的持久化。

#### 9.2 AOF持久化

**`AOF（Append Only File）`**，解决了数据持久化的实时性的问题，是目前Redis持久化的主流方式。AOF的工作流程包括：命令写入（append）、文件同步（sync）、文件重写（rewrite）、重启加载（load）,如下图：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722012443186.png" alt="image-20220722012443186" style="zoom:50%;" />

开启 AOF 持久化后每执⾏⼀条 **写命令**， Redis 就会将该命令写⼊硬盘中
的 AOF ⽂件。 AOF ⽂件的保存位置和 RDB ⽂件的位置相同，

都是通过 dir 参数设置的，默认的
⽂件名是 appendonly.aof 。



AOF默认不开启，需要修改配置项来启用它：

```properties
# 是否开启AOF功能，默认是no
appendonly yes         
# AOF文件的名称
appendfilename "appendonly.aof"
```

在 Redis 的配置⽂件中存在三种不同的 AOF 持久化⽅式，它们分别是：  

```properties
# 每次有数据修改发⽣时都会写⼊AOF⽂件,这样会严重降低Redis的速度
appendfsync always 
# 写命令执行完先放入AOF缓冲区，然后表示每隔1秒将缓冲区数据写到AOF文件，是默认方案
appendfsync everysec
# 写命令执行完先放入AOF缓冲区，由操作系统决定何时将缓冲区内容写回磁盘
appendfsync no
```

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220608191431839.png" alt="image-20220608191431839" style="zoom:67%;" />

AOF持久化的优缺点如下：

- 优点：与RDB持久化可能丢失大量的数据相比，AOF持久化的安全性要高很多。通过使用`appendfsync everysec`选项，用户可以将数据丢失的时间窗口限制在1秒之内。
- 缺点：AOF文件存储的是协议文本，它的体积要比二进制格式的`.rdb`文件大很多。AOF需要通过执行AOF文件中的命令来恢复数据库，其恢复速度比RDB慢很多。AOF在进行重写时也需要创建子进程，在数据库体积较大时将占用大量资源，会导致服务器的短暂阻塞。



##### 补充内容： AOF 重写（rewrite）

​	AOF 重写可以产⽣⼀个新的 AOF ⽂件，这个新的 AOF ⽂件和原有的 AOF ⽂件所保存的数据库
状态⼀样，但体积更⼩。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722012850628.png" alt="image-20220722012850628" style="zoom: 67%;" />

​	AOF 重写是⼀个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序⽆须对现有
AOF ⽂件进⾏任何读⼊、分析或者写⼊操作。

​	在执⾏ `BGREWRITEAOF` 命令时， Redis 服务器会维护⼀个 AOF 重写缓冲区，该缓冲区会在⼦进程创建新 AOF ⽂件期间，记录服务器执⾏的所有写命令。当⼦进程完成创建新 AOF ⽂件的⼯作之后，服务器会将重写缓冲区中的所有内容追加到新 AOF ⽂件的末尾，使得新旧两个 AOF ⽂件所保存的数据库状态⼀致。最后，服务器⽤新的 AOF ⽂件替换旧的 AOF ⽂件，以此来完成
AOF ⽂件重写操作 

​	Redis 也会在触发阈值时自动去重写AOF文件。阈值也可以在redis.conf中配置： 

```conf
# AOF文件比上次文件 增长超过多少百分比则触发重写
auto-aof-rewrite-percentage 100
# AOF文件体积最小多大以上才触发重写 
auto-aof-rewrite-min-size 64mb 
```



#### 9.3 RDB-AOF混合持久化：

Redis从4.0开始引入RDB-AOF混合持久化模式，这种模式是基于AOF持久化构建而来的。用户可以通过配置文件中的

`aof-use-rdb-preamble yes`配置项开启AOF混合持久化。Redis服务器在执行AOF重写操作时，会按照如下原则处理数据：

- 像执行BGSAVE命令一样，根据数据库当前的状态生成相应的RDB数据，并将其写入AOF文件中；
- 对于重写之后执行的Redis命令，则以协议文本的方式追加到AOF文件的末尾，即RDB数据之后。通过使用RDB-AOF混合持久化，用户可以同时获得RDB持久化和AOF持久化的优点，服务器既可以通过AOF文件包含的RDB数据来实现快速的数据恢复操作，又可以通过AOF文件包含的AOF数据来将丢失数据的时间窗口限制在1s之内。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220608191640609.png" alt="image-20220608191640609" style="zoom:67%;" />



### 10.什么是主从模式？（非重点，理解记忆）

==参考答案==

> 其实 Redis 的主从模式很简单，在实际的生产环境中很少使用，不建议在实际的生产环境中使用主从模式来提供系统的高可用性，之所以不建议使用都是由它的缺点造成的，在数据量非常大的情况，或者对系统的高可用性要求很高的情况下，主从模式也是不稳定的。虽然这个模式很简单，但是这个模式是其他模式的基础，所以理解了这个模式，对其他模式的学习会很有帮助。

​	单节点Redis的并发能力是有上限的，要进一步提高Redis的并发能力，就需要搭建主从集群，实现<font color="red">**读写分离**</font>。

​	Redis 多机器部署时，这些机器节点会被分成两类，一类是**主节点（Master 节点）**，一类是**从节点（Slave 节点）**。一般主节点进行写操作，而从节点只能进行读操作。当主节点的数据发生变化时，会将变化的数据同步给从节点，这样从节点的数据就可以和主节点的数据保持一致了。一个主节点可以有多个从节点，但是一个从节点会只会有一个主节点，也就是所谓的**一主多从**结构。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722013359046.png" alt="image-20220722013359046" style="zoom: 67%;" />

（1）文件配置方式如下：

```properties
# 配置主节点的ip和端口
slaveof 192.168.1.10 6379
# 从redis2.6开始，从节点默认是只读的
Slave -read-only yes
# 假设主节点有登录密码，是123456
masterauth 123456
```

（2）或者通过redis-cli配置

```shell
127.0.0.1:6379> redis-server --slaveof 127.0.0.1 7001 (redis:6379设置为redis:7001的从节点)
```

#### 10.1  Redis的主从同步是如何实现的？

==参考答案==

Redis使用`psync`命令完成主从同步，同步过程分为**全量同步**和**增量同步**。**全量同步**一般用于初次同步的场景，**增量同步**则用于处理因网络中断等原因造成数据丢失的场景。`psync`命令需要以下参数的支持：

1. **`[replicationid]`**：简称 replid，数据集的标记，id 一致则说明是同一数据集。每一个 master 都有唯一的replid，slave 则会继承 master 节点的 replid 。
2. **`[offset]`**：同步偏移量，随着记录在 repl_baklog 中的数据增多而逐渐增大。slave 完成同步时也会记录当前同步的 offset。如果 slave 的 offset 小于 master 的 offset，说明 slave 数据落后于 master，需要更新。

#### 10.2 Redis如何进行全量同步？

流程：

> replicaof 命令等同于 slaveof 命令

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220608194346413.png" alt="image-20220608194346413" style="zoom: 50%;" />

这里有一个问题，Master 如何得知salve是第一次来连接呢？？

有几个概念，可以作为判断依据：

- **`replication id`**：简称replid，是数据集的标记，id一致则说明是同一数据集。每一个Master 都有唯一的replid，Slave 则会继承Master 节点的replid
- **`offset`**：偏移量，随着记录在repl_baklog中的数据增多而逐渐增大。Slave 完成同步时也会记录当前同步的offset。如果Slave 的offset小于Master 的offset，说明Slave 数据落后于Master ，需要更新。

因此Slave 做数据同步，必须向Master 声明自己的replication id 和offset，Master 才可以判断到底需要同步哪些数据。

因为Slave 原本也是一个Master ，有自己的replid和offset，当第一次变成Slave ，与Master 建立连接时，发送的replid和offset是自己的replid和offset。Master 判断发现Slave 发送来的replid与自己的不一致，说明这是一个全新的Slave ，就知道要做全量同步了。

Master 会将自己的replid和offset都发送给这个Slave ，Slave 保存这些信息。以后Slave 的replid就与Master 一致了。

因此，**Master 判断一个节点是否是第一次同步的依据，就是看replid是否一致**。

完整流程描述：

- Slave 节点请求增量同步
- Master 节点判断replid，发现不一致，拒绝增量同步
- Master 将完整内存数据生成RDB，发送RDB到Slave 
- Slave 清空本地数据，加载Master 的RDB
- Master 将RDB期间的命令记录在repl_baklog，并持续将log中的命令发送给Slave 
- Slave 执行接收到的命令，保持与Master 之间的同步

#### 10.3 Redis如何进行增量同步？

全量同步需要先做RDB，然后将RDB文件通过网络传输给Slave ，成本太高了。因此除了第一次做全量同步，其它大多数时候Slave 与Master 都是做**增量同步**。

什么是增量同步？就是只更新Slave 与Master 存在差异的部分数据。如图：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722013626767.png" alt="image-20220722013626767" style="zoom:50%;" />



<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220608220422392.png" alt="image-20220608220422392" style="zoom:50%;" />



#### 10.4 什么时候执行全量同步？

- Slave 节点第一次连接Master 节点时
- Slave 节点断开时间太久，`repl_baklog`中的 offset 已经被覆盖时

#### 10.5 什么时候执行增量同步？

- Slave 节点断开又恢复，并且在`repl_baklog`中能找到 offset 时

#### 10.6 如何优化主从同步？

主从同步可以保证主从数据的一致性，非常重要。

可以从以下几个方面来优化Redis主从集群：

- 在Master 中配置`repl-diskless-sync yes`启用无磁盘复制（即直接写入网络流中），避免全量同步时的磁盘IO。
- Redis单节点上的内存占用不要太大，减少RDB导致的过多磁盘IO
- 适当提高`repl_baklog`的大小，发现 Slave 宕机时尽快实现故障恢复，尽可能避免全量同步
- 限制一个 Master 上的 Slave 节点数量，如果实在是太多 Slave ，则可以采用**主-从-从链式结构**，减少 Master 压力

主从从架构图：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722013757744.png" alt="image-20220722013757744" style="zoom:50%;" />



#### 10.7 主从模式的优缺点

##### 优点

- 支持主从复制，Master 能自动将数据同步到 Slave；
- 为了分载 Master 的读操作压力，Slave 服务器可以为客户端提供只读操作的服务，写服务由 Master 来完成，可以进行**读写分离**；
- Slave 同样可以接受其他 Slaves 的连接和同步请求，即可以采用**主-从-从链式结构**，这样可以有效地分载 Master 的同步压力;
- Master 和 Slave 都是以非阻塞的方式完成数据同步。所以在 Master-Slave 同步期间，客户端仍然可以提交查询或修改请求；

##### 缺点

- 不具备自动容错和恢复功能，Master 或 Slave 的宕机都可能导致客户端读写请求失败，需要等待机器重启或者手动切换客户端的 IP 才能恢复；
- Master 宕机，如果宕机前数据没有同步完，则切换IP后会存在数据不一致的问题，降低了系统的可用性；
- 如果多个 Slave 断线了，需要重启的时候，尽量不要在同一时间段进行重启。因为只要 Slave 启动，就会发送 psync 请求和主机全量同步，当多个 Slave 重启的时候，可能会导致 Master IO 剧增从而宕机；
- 难以支持在线扩容，Redis的容量受限于单机配置。

### 11.说⼀下你对哨兵模式的理解？

==参考答案==  

​	在主从模式下，Redis 同时提供了哨兵命令`redis-Sentinel `。

哨兵模式是在主从模式的基础上增加了哨兵机制，哨兵作为一个独立的进程运行，其原理是哨兵进程向所有的 Redis 机器发送命令，等待 Redis 服务器响应，从而监控运行的多个 Redis 实例。

​	Sentinel 可以有多个，一般为了便于决策选举，使用奇数个 Sentinel 。Sentinel 可以和 Redis 机器部署在一起，也可以部署在其他的机器上。多个 Sentinel 构成一个 Sentinel 集群， Sentinel 也会相互通信，检查 Sentinel 是否正常运行。如果发现 Master 宕机，Sentinel 之间会进行决策选举新的 Master

哨兵的结构如图：

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220608224829810.png" alt="image-20220608224829810" style="zoom: 67%;" />



#### 11.1 哨兵是如何监控集群的？

Sentinel基于**心跳机制**监测服务状态，每隔1秒向集群的每个 Redis 实例发送`ping`命令：

- **主观下线**：如果某 Sentinel 节点发现某实例未在规定时间响应，则认为该实例**主观下线**。

- **客观下线**：若超过指定数量（quorum）的 Sentinel 都认为该实例主观下线，则该实例**客观下线**。quorum 值最好超过Sentinel 实例数量的一半。

#### 11.2 哨兵的作用如下：

- **监控**：Sentinel 会基于心跳机制监控 Redis 集群中的每一台 Redis 实例的运行状态

- **自动故障恢复**：如果 Master 故障，Sentinel 会选举一个 Slave 提升为 Master。当故障实例恢复后也以新的 Master 为主

- **通知**：Sentinel 充当 Redis 客户端的服务发现来源，当集群发生故障转移时，会将最新信息推送给Redis的客户端

#### 11.3 哨兵是如何完成集群故障恢复的？

一旦发现 Master 故障，Sentinel 需要在 Slave 中选择一个作为新的 Master ，选择依据是这样的：

- 首先会判断Slave 节点与Master 节点断开时间长短，如果超过指定值（down-after-milliseconds * 10）则会排除该Slave 节点
- 然后判断 Slave 节点的`slave-priority`值，越小优先级越高，如果是0则永不参与选举
- 如果`slave-prority`一样，则判断 Slave 节点的 offset 值，越大说明数据越新，优先级越高
- 最后是判断 Slave 节点的运行id大小，越小优先级越高。

#### 11.4 当选出一个新的Master 后，该如何实现切换呢？

流程如下：

- Sentinel 给备选的 Slave 1节点发送`slaveof no one`命令，让该节点成为 Master 
- Sentinel 给所有其它 Slave 发送`slaveof [ip] [port]` 命令，让这些 Slave 成为新 Master 的从节点，开始从新的Master 上同步数据。
- 最后，Sentinel 将故障节点标记为 Slave ，当故障节点恢复后会自动成为新的 Master 的 Slave 节点

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220608230327405.png" alt="image-20220608230327405" style="zoom:67%;" />





#### 11.5 哨兵模式的优缺点

##### 优点

- 基于**主从模式**，拥有主从模式的所有优点，如支持主从复制，能自动进行主从同步；可以进行读写分离；可以采用**主-从-从链式结构**，从而有效地分载 Master 的同步压力；Master 和 Slave 都是以非阻塞的方式完成数据同步，所以在 Master-Slave 同步期间，客户端仍然可以提交查询或修改请求。
- **哨兵模式**下，支持自动故障恢复，系统可用性更高

##### 缺点

- 如果多个 Slave 断线了，需要重启的时候，尽量不要在同一时间段进行重启。因为只要 Slave 启动，就会发送 sync 请求和主机全量同步，当多个 Slave 重启的时候，可能会导致 Master IO 剧增从而宕机；
- <font color="red">难以支持在线扩容，Redis的容量受限于单机配置</font>
- <font color="red">每台机器上的数据是一样的，内存的复用性较低，同时需要额外的资源来启动 Sentinel 进程，实现相对复杂一点</font>

### 12. 说一下你对集群模式（Redis Cluster）的理解？

==参考答案==

​	Redis 的哨兵模式基本已经可以实现高可用，读写分离 ，但是在这种模式下每台 Redis 服务器都存储相同的数据，很浪费内存，而且不支持在线扩容，难以解决<u>海量数据存储问题</u>和<u>高并发写的问题</u>。所以在 redis3.0 上加入了 Cluster 集群模式，实现了 Redis 的分布式存储，对数据进行分片，也就是说每台 Redis 节点上存储不同的内容。

#### 12.1 分片集群特征是什么？

> 可以理解为将哨兵与master合为一体了

- 集群中有多个 Master，每个 Master 保存不同数据；每个 Master 又可以有多个 Slave 节点。每个节点都会通过集群总线(cluster bus)，与其他的节点进行通信（完全图）
- Master 之间通过心跳机制`ping`监测彼此健康状态
- Redis-Cluster模式 采用去中心化的思想，没有中心节点的说法，客户端与 Redis 节点直连，不需要连接集群所有节点，连接集群中任何一个可用节点最终都会被转发到正确节点（路由转发）
- 为了保证高可用，Cluster 模式也引入主从复制模式，一个主节点对应一个或者多个从节点。如果半数以上的 Master 与 Master 1 通信超时，那么认为 Master 1 宕机了，就会启用 Master 1 的从节点 Slave 1，将 Slave 1 变成主节点继续提供服务。
- 如果 至少一个Master  和它的所有 Slave 都宕机了，整个集群就会进入 `fail` 状态。因为集群的 slot 映射不完整。如果集群超过半数以上的 Master 挂掉，无论是否有 Slave，集群都会进入 `fail` 状态。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722014028453.png" alt="image-20220722014028453" style="zoom: 67%;" />

#### 12.2 说说 Redis 哈希槽的概念？
Redis 集群模式没有使用一致性 hash,而是引入了哈希槽的概念， Redis 集群有 16384 (0~16383) 个哈希槽，每个 key
通过 CRC16 校验后对 16384 取模来决定放置哪个槽，Redis会把每一个master节点映射到16384个插槽（hash slot）上，集群的每个节点负责一部分 hash 槽。  这就意味着Redis 集群最大节点个数是
**16384** 个  

> 一致性hash https://developer.huawei.com/consumer/cn/forum/topic/0203810951415790238?fid=0101592429757310384

#### 12.3 Redis 集群会有写操作丢失吗？为什么？
Redis 并不能保证数据的强一致性，这意味这在实际中集群在特定的条件下可能会丢失写操作。  

#### 12.4 集群扩缩容（集群伸缩）

对 redis 集群的扩容就是向集群中添加机器，缩容就是从集群中删除机器；并重新将 16384 个 slots 分配到集群中的节点上（即数据迁移）。

- **扩容（add-nodes）**时，先使用`redis-cli --cluster add-node`将新的机器加到集群中，这时新机器虽然已经在集群中了，但是没有分配 slots，依然是不起做用的。在使用 `redis-cli --cluster  reshard`进行分片重哈希（插槽转移），将旧节点上的 部分slots 转移到新节点上后，新节点才能起作用。

- **缩容（del-nodes）**时，先要使用 ` redis-cli --cluster reshard`将删除的机器上的 slots转移到其他机器上，然后使用`redis-cli --cluster del-nodes`将机器从集群中删除。

#### 12.5 集群模式的优缺点

##### 优点

- 可扩展性：支持在线扩容，节点可动态添加或删除;
- 基于**主从模式**，拥有主从模式的所有优点，如支持主从复制，能自动进行主从同步；可以进行读写分离。
- 每个 Master 的数据都不一样，内存的可用性较高，支持海量数据的存储，缓解了高并发写的问题。

##### 缺点

- Redis Cluster 是无中心节点的集群架构，依靠 Goss 协议(谣言传播)协同自动化修复集群的状态。但 GossIp 有消息延时和消息冗余的问题，在集群节点数量过多的时候，节点之间需要不断进行 PING/PANG 通讯，不必要的流量占用了大量的网络资源。虽然 Reds4.0 对此进行了优化，但这个问题仍然存在。
- 数据迁移问题：Redis Cluster 可以进行节点的动态扩容缩容，这一过程，在目前实现中，还处于半自动状态，需要人工介入。在扩缩容的时候，需要人为进行数据迁移。

### 13. 如何实现Redis的高可用？

==参考答案==

> 所谓的高可用，也叫 HA（High Availability），是分布式系统架构设计中必须考虑的因素之一，它通常是指，通过设计减少系统不能提供服务的时间。如果在实际生产中，如果 redis 只部署一个节点，当机器故障时，整改服务都不能提供服务了。这就是我们常说的单点故障。如果 redis 部署了多台，当一台或几台故障时，整个系统依然可以对外提供服务，这样就提高了服务的可用性。

redis 高可用的两种模式：**哨兵模式**，**集群模式**。（把上面9和10详细说一下）

### 14. 如何保证缓存与数据库的双写一致性？

3种常用的缓存读写策略：

#### 14.1 Cache Aside Pattern（旁路缓存模式）

由缓存的调用者，再更新数据库的同时更新缓存



**Cache Aside Pattern 是我们平时使用比较多的一个缓存读写模式，比较适合读请求比较多的场景。**

Cache Aside Pattern 中服务端需要同时维系 DB 和 cache，并且是以 DB 的结果为准。



> <img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722015759577.png" alt="image-20220722015759577" style="zoom:50%;" />

- **读操作**：
  - 缓存命中则直接返回
  - 缓存未命中则查询数据库，并写入缓存，设定超时时间

- **写操作**：
  - **先写数据库，然后再删除缓存**
  - 要确保数据库与缓存操作的原子性



面试官很可能会追问：“**在写数据的过程中，可以先删除 cache ，后更新 DB 么？**”

**答案：** 那肯定是不行的！因为这样可能会造成**数据库（DB）和缓存（Cache）数据不一致**的问题。为什么呢？比如说请求1 先写数据A，请求2随后读数据A的话就很有可能产生数据不一致性的问题。这个过程可以简单描述为：

> 请求1先把cache中的A数据删除 -> 请求2缓存未命中，从DB中读取数据->请求1再把DB中的A数据更新。

当你这样回答之后，面试官可能会紧接着就追问：“**在写数据的过程中，先更新DB，后删除cache就没有问题了么？**”

**答案：** 理论上来说还是可能会出现数据不一致性的问题，不过概率非常小，**因为缓存的写入速度是比数据库的写入速度快很多**！

比如请求1先读数据 A，请求2随后写数据A，并且数据A不在缓存中的话也有可能产生数据不一致性的问题。这个过程可以简单描述为：

> 请求1从DB读数据A->请求2写更新数据 A 到数据库并把删除cache中的A数据->请求1将数据A写入cache。



现在我们再来分析一下 **Cache Aside Pattern 的缺陷**。

**缺陷1：首次请求数据一定不在 cache 的问题**

解决办法：可以将热点数据可以提前放入cache 中。

**缺陷2：写操作比较频繁的话导致cache中的数据会被频繁被删除，这样会影响缓存命中率 。**

解决办法：

- 数据库和缓存数据强一致场景 ：更新DB的时候同样更新cache，不过我们需要加一个锁/分布式锁来保证更新cache的时候不存在线程安全问题。
- 可以短暂地允许数据库和缓存数据不一致的场景 ：更新DB的时候同样更新cache，但是给缓存加一个比较短的过期时间，这样的话就可以保证即使数据不一致的话影响也比较小。

#### 14.2 Read/Write Through Pattern（读写穿透）

缓存与数据库整合为一个服务，由服务来维护一致性。调用者调用该服务，无需关心缓存一致性问题。



Read/Write Through Pattern 中服务端把 cache 视为主要数据存储，从中读取数据并将数据写入其中。cache 服务负责将此数据读取和写入 DB，从而减轻了应用程序的职责。

这种缓存读写策略在开发过程中非常少见。抛去性能方面的影响，大概率是因为我们经常使用的分布式缓存 Redis 并没有提供 cache 将数据写入DB的功能。

- **写（Write Through）：**
  - 先查 cache，cache 中不存在，直接更新 DB。
  - cache 中存在，则先更新 cache，然后 cache 服务自己更新 DB（**同步更新 cache 和 DB**）。

- **读(Read Through)：**
  - 从 cache 中读取数据，读取到就直接返回 。
  - 读取不到的话，先从 DB 加载，写入到 cache 后返回响应。



Read-Through Pattern 实际只是在 Cache-Aside Pattern 之上进行了封装。在 Cache-Aside Pattern 下，发生读请求的时候，如果 cache 中不存在对应的数据，是由客户端自己负责把数据写入 cache，而 Read-Through Pattern 则是 cache 服务自己来写入缓存的，这对客户端是透明的。

和 Cache Aside Pattern 一样， Read-Through Pattern 也有首次请求数据一定不再 cache 的问题，对于热点数据可以提前放入缓存中。

#### 14.3 Write Behind Pattern（异步缓存写入）

Write Behind Pattern 和 Read/Write Through Pattern 很相似，两者都是由 cache 服务来负责 cache 和 DB 的读写。

但是，两个又有很大的不同：**Read/Write Through 是同步更新 cache 和 DB，而 Write Behind Caching 则是只更新缓存，不直接更新 DB，而是改为异步批量的方式来更新 DB。**



很明显，这种方式对数据一致性带来了更大的挑战，比如cache数据可能还没异步更新DB的话，cache服务可能就就挂掉了。

这种策略在我们平时开发过程中也非常非常少见，但是不代表它的应用场景少，比如消息队列中消息的异步写入磁盘、MySQL 的 InnoDB Buffer Pool 机制都用到了这种策略。



Write Behind Pattern 下 DB 的写性能非常高，非常适合一些数据经常变化又对数据一致性要求没那么高的场景，比如浏览量、点赞量。

### 15. 缓存穿透、缓存击穿、缓存雪崩有什么区别，该如何解决？

==参考答案==

#### 15.1 缓存穿透

**缓存穿透**是指客户端**请求的数据在缓存中和数据库中都不存在**，这样缓存永远不会生效，这些请求都会达到数据库，导致其负载过大，甚至宕机。出现这种情况的原因，可能是业务层误将缓存和库中的数据删除了，也可能是有人恶意攻击，专门访问库中不存在的数据。



解决方案：

- **缓存空对象**：数据库第一次未命中后，将空值存入缓存层，客户端再次访问数据时，缓存层会直接返回空值。

- **布隆过滤器**：将数据存入布隆过滤器，访问缓存之前以过滤器拦截，若请求的数据不存在则直接返回空值。

  **它的优点是空间效率和查询时间都比一般的算法要好的多，缺点是有一定的误识别率和删除困难。**布隆过滤器可以检查值是 **“可能在集合中”** 还是 **“绝对不在集合中”**

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722121922522.png" alt="image-20220722121922522" style="zoom: 67%;" />

#### 15.2 缓存雪崩

**缓存雪崩**是指在同一时段大量的缓存 key 同时失效（过期）或者 Redis 服务宕机，导致大量请求直达数据库，造成数据库宕机。



解决方案：

1. **避免数据同时过期**：设置 key 的 TTL 时，附加一个随机数，避免大量的 key 同时过期。
2. **构建高可用的 Redis 服务**：采用哨兵或集群模式，部署多个Redis实例，个别节点宕机，依然可以保持服务的整体可用。
3. **启用降级限流和熔断措施**：在发生雪崩时，若应用访问的不是核心数据，则直接返回预定义信息/空值/错误信息。或者在发生雪崩时，对于访问缓存的请求，客户端并不会把请求发给 Redis，而是直接返回。

#### 15.3 缓存击穿

**缓存击穿问题**也叫热点Key问题，就是一个被**高并发访问**的key突然失效了，在其缓存失效的瞬间，大量请求直达存储层，导致服务崩溃。



解决方案：

1. **逻辑过期**：为热点数据设置逻辑过期时间，当发现该数据逻辑过期时，使用单独的线程重建缓存。
2. **加互斥锁**：对数据的访问加互斥锁，当一个线程访问该数据时，其他线程只能等待。这个线程访问过后，缓存中的数据将被重建，届时其他线程就可以直接从缓存中取值。

<img src="https://raw.githubusercontent.com/ayifuture0920/Img/main/pictures/image-20220722152131729.png" alt="image-20220722152131729" style="zoom: 50%;" />

### 16. 说一说你对布隆过滤器的理解

==参考答案==

> **布隆过滤器**原理：
>
> **当一个元素加入布隆过滤器中的时候，会进行哪些操作：**
>
> 1. 使用布隆过滤器中的多个哈希函数对元素值进行计算，得到多个哈希值（有几个哈希函数得到几个哈希值）。
> 2. 根据得到的哈希值，在位数组中把对应多个下标的值置为 1。
>
> 我们再来看一下，**当我们需要判断一个元素是否存在于布隆过滤器的时候，会进行哪些操作：**
>
> 1. 对给定元素再次进行相同的哈希计算；
> 2. 得到值之后判断位数组中的多个哈希值对应的位置是否都为 1，如果值都为 1，那么说明这个值在布隆过滤器中；如果存在一个值不为 1，说明该元素不在布隆过滤器中。
>
> 然后，一定会出现这样一种情况：**不同的字符串可能哈希出来的位置相同。** （可以适当增加位数组大小或者调整我们的哈希函数来降低概率）
>
> *参考：[布隆过滤器你值得拥有的开发利器](https://segmentfault.com/a/1190000021136424)*

​		布隆过滤器可以用很低的代价，估算出数据是否真实存在。例如：给用户推荐新闻时，要去掉重复的新闻，就可以利用布隆过滤器，判断该新闻是否已经推荐过。

布隆过滤器的核心包括两部分：

1. 一个大型的位数组；
2. 若干个不一样的哈希函数，每个哈希函数都能将哈希值算的比较均匀。

布隆过滤器的工作原理：

1. 添加key时，每个哈希函数都利用这个key计算出一个哈希值，再根据哈希值计算一个位置，并将位数组中这个位置的值设置为1。
2. 询问key时，每个哈希函数都利用这个key计算出一个哈希值，再根据哈希值计算一个位置。然后对比这些哈希函数在位数组中对应位置的数值：
   - 如果这几个位置中，有一个位置的值是0，就说明这个布隆过滤器中，不存在这个key。
   - 如果这几个位置中，所有位置的值都是1，就说明这个布隆过滤器中，极有可能存在这个key。之所以不是百分之百确定，是因为也可能是其他的key运算导致该位置为1。

### 17. 说说Redis中List结构的相关操作

==参考答案==

列表是线性有序的数据结构，它内部的元素是可以重复的，并且一个列表最多能存储$2^{32}-1$个元素。列表包含如下的常用命令：

- `LPUSH key element ...` ：向列表左侧插入一个或多个元素
- `LPOP key`：移除并返回列表左侧的第一个元素，没有则返回nil
- `RPUSH key element ... `：向列表右侧插入一个或多个元素
- `RPOP key`：移除并返回列表右侧的第一个元素
- `LRANGE key star end`：返回一段角标范围内的所有元素
- `BLPOP和BRPOP`：与LPOP和RPOP类似，只不过在没有元素时等待指定时间，而不是直接返回nil

### 18. 你要如何设计Redis的过期时间？

==参考答案==

1. 热点数据不设置过期时间，使其达到 “物理”上的永不过期，可以避免缓存击穿问题；
2. 在设置过期时间时，可以附加一个随机数，避免大量的 key 同时过期，导致缓存雪崩。

### 19. 说一说hash类型底层的数据结构

### 20. 介绍一下zset类型底层的数据结构

### 21. 如何利用Redis实现分布式Session？

### 22. Redis 性能优化

#### 22.1 Redis bigkey

##### 22.1.1 什么是 bigkey？

简单来说，如果一个 key 对应的 value 所占用的内存比较大，那这个 key 就可以看作是 bigkey。具体多大才算大呢？有一个不是特别精确的参考标准：string 类型的 value 超过 10 kb，复合类型的 value 包含的元素超过 5000 个（对于复合类型的 value 来说，不一定包含的元素越多，占用的内存就越多）。

##### 22.1.2 bigkey 有什么危害？

除了会消耗更多的内存空间，bigkey 对性能也会有比较大的影响。

因此，我们应该尽量避免写入 bigkey！

##### 22.1.3 如何发现 bigkey？

**1、使用 Redis 自带的 `--bigkeys` 参数来查找。**

```bash
# redis-cli -p 6379 --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far '"ballcat:oauth:refresh_auth:f6cdb384-9a9d-4f2f-af01-dc3f28057c20"' with 4437 bytes
[00.00%] Biggest list   found so far '"my-list"' with 17 items

-------- summary -------

Sampled 5 keys in the keyspace!
Total key length in bytes is 264 (avg len 52.80)

Biggest   list found '"my-list"' has 17 items
Biggest string found '"ballcat:oauth:refresh_auth:f6cdb384-9a9d-4f2f-af01-dc3f28057c20"' has 4437 bytes

1 lists with 17 items (20.00% of keys, avg size 17.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
4 strings with 4831 bytes (80.00% of keys, avg size 1207.75)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00
```

从这个命令的运行结果，我们可以看出：这个命令会扫描(Scan) Redis 中的所有 key ，会对 Redis 的性能有一点影响。并且，这种方式只能找出每种数据结构 top 1 bigkey（占用内存最大的 string 数据类型，包含元素最多的复合数据类型）。

**2、分析 RDB 文件**

通过分析 RDB 文件来找出 big key。这种方案的前提是你的 Redis 采用的是 RDB 持久化。

网上有现成的代码/工具可以直接拿来使用：

- [redis-rdb-tools](https://github.com/sripathikrishnan/redis-rdb-tools) ：Python 语言写的用来分析 Redis 的 RDB 快照文件用的工具
- [rdb_bigkeys](https://github.com/weiyanwei412/rdb_bigkeys) : Go 语言写的用来分析 Redis 的 RDB 快照文件用的工具，性能更好。

#### 22.2 大量 key 集中过期问题

我在上面提到过：对于过期 key，Redis 采用的是 **定期删除+惰性/懒汉式删除** 策略。

定期删除执行过程中，如果突然遇到大量过期 key 的话，客户端请求必须等待定期清理过期 key 任务线程执行完成，因为这个这个定期任务线程是在 Redis 主线程中执行的。这就导致客户端请求没办法被及时处理，响应速度会比较慢。

如何解决呢？下面是两种常见的方法：

1. 给 key 设置随机过期时间。
2. 开启 lazy-free（惰性删除/延迟释放） 。lazy-free 特性是 Redis 4.0 开始引入的，指的是让 Redis 采用异步方式延迟释放 key 使用的内存，将该操作交给单独的子线程处理，避免阻塞主线程。

个人建议不管是否开启 lazy-free，我们都尽量给 key 设置随机过期时间。

