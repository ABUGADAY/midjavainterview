# Redis

## 1. 什么是Redis？

Redis 是一个基于 **内存** 的高性能 **key-value** 数据库。



##  2. Redis的特点

Redis 本质上是一个 key-value 类型的内存数据库，很像 memcached, 整个数据库统统加载在内存当中进行操作，定期通过 **异步** 操作把数据库数据 flush 到硬盘上进行保存。因为是纯内存操作，Redis 的性能非常出色，每秒可以处理超过10万次读写操作，是已知性能最快的 key-value DB。

Redis 的出色之处不仅仅是性能，Redis最大的魅力是支持保存**多种数据结构** ，此外单个 value 的最大限制是**1G** ，不像 memcached 只能保存 1MB 的数据，因此 Redis 可以用来实现很多有用的功能，比方说他的 List 来做 **FIFO双向链表** 实现一个轻量级的高性能消息队列服务，用他的 Set 可以作高性能的 **tag系统**等等。另外 Redis 也可以对存入的 key-value 设置 expire 时间，因此也可以被当作一个功能加强版的 memcached 来用。

Redis 的主要确定是数据库容量**受到物理内存的限制**，不能用作海量数据的高性能读写，因此Redis时和的场景主要局限在**较小数据量的高性能操作和运算上** 。



## 3. 使用Redis有哪些好处？

1. **速度快**，因为数据存在内存中，类似HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；
2. **支持丰富数据类型** ，支持String，List，Set， SortedSet,Hash;
3. **支持事务** ，操作都是原子性，所谓的原子性 就是对书u的更改要么全部执行，要么全部不执行；
4. **丰富的特性**，可用于缓存，消息，按key设置过期时间，过期后将会自动删除。



## 4.Redis相比memcached有哪些优势？

1. memcached所有值均是简单的字符串，redis作为其替代者，支持**更为丰富的数据类型**；
2. redis的**速度**比memecached块很多；
3. redis可以**持久化其数据**；



##  5.Memcache与Redis的区别都有哪些？

1. 存储方式：Memecache把数据全部存在内存之中，断点后会刮掉，数据不能超过内存大小。Redis有部分存在硬盘上，这样能保证数据的**持久性**。
2. 数据支持类型：Memcache对数据类型支持相对简单。Redis有**复杂的数据类型**。
3. 使用底层模型不同：它们之间底层实现方式以及与客户端之间通信的应用协议不一样。Redis直接自己构建了 VM 机制，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求。



##  6.Redis常见性能问题和解决方案

1. Master写内存快照：save命令 调度rdbSave函数，会阻塞主线程的工作，当快照比较大时对性能影响是非常大的，会间歇性暂停服务，所以Master最好不要写内存快照。
2. Master AOF持久化：如果不重写AOF日志文件，这个持久化方式对性能的影响是最小的，但是AOF文件会不断增大，AOF文件过大会影响Master重启的恢复速度。Master最好不要做任何持久化工作，包括内存快照和AOF日志文件，特别是不要启动内存快照做持久化，如果数据比较关键，某个Slave开启AOF备份数据，策略为每秒同步一次。
3. Master调哦用BGREWITEAOF 重写AOF文件：AOF在重写的时候会占大量的CPU和内存资源，导致服务load过高，出现短暂服务暂停现象。
4. Redis主从复制的性能问题：为了主从复制的速度和连接的稳定性，Slave和Master最好在同一个局域网内。



##  7.MySQL里有2000w数据redis中只存20w的数据，如何保证Redis中的数据都是热点数据

相关知识：Redis内存数据集上升到一定大小的时和，就会施行数据淘汰策略(回收策略)。Redis提供6中数据淘汰策略：

- volatile-lru: 从已设置过期时间的数据集(server.db[i].expires)中挑选最近最少使用的数据淘汰
- volatile-ttl: 从已设置过期时间的数据集(server.db[i].expires)中选择将要过期的数据淘汰
- volatile-random: 从已设置过期时间的数据集(server.db[i].expires)中任意选择数据淘汰
- allkeys-lru: 从数据集(server.db[i].dict)中挑选最近最少使用的数据淘汰
- allkeys-random: 从数据集(server.db[i].dict)中任意选择数据淘汰
- no-envictio(驱逐):禁止驱逐数据



## 8.请使用Redis和任意语言实现一段恶意登陆保护的代码，限制1小时内每用户Id最多只能登陆5次。具体登陆函数或功能用空函数即可，不用详细写出。

用列表实现: 列表中每个元素代表登陆时间，只要最后5此登陆时间和现在时间差不超过一小时就禁止登陆 。



## 9.为什么Redis需要把所有数据放到内存中？

Redis为了达到最快的读写速度将数据都读到内存中，并通过异步的方式将数据写入磁盘。所以Redis具有快速和数据持久化的特征。如果部将数据放在内存中，磁盘I/O速度会严重影响Redis的性能。在内存越来越便宜的今天，Redis将会越来越受欢迎。

如果设置了最大使用的内存，则数据已有记录数达到内存限制后不能继续插入新值。



## 10.Redis是单进程单线程的。

Redis利用队列技术将并发访问变为串行访问，消除了传统 数据库串行控制的开销



## 11.Redis的并发竞争问题如何解决

Redis为单进程单线程模式，采用队列模式将并发访问变为串行访问。Redis本身没有锁的概念，Redis对于多个客户端连接并不存在竞争，但是Jedis客户端对Redis进行并发访问是会发生连接超时、数据转换错误、阻塞、客户端关闭连接等问题，这些问题均是由于客户端连接混乱造成。对此有2种解决方法：

1. 客户端角度，为保证每个苦湖段间正常有序与Redis进行通信，对连接进行池化，同时对客户端读写Redis操作采用内部锁synchronized。
2. 服务器角度，利用setnx实现锁。

注：对于第一种，需要应用程序自己处理资源的同步，可以使用的方法比较通俗，可以使用synchronized也可以使用lock；第二种需要用到Redis的setnx命令，但是需要注意一些问题。



## 12.redis事务的了解 CAS(check-and-set操作实现乐观锁)？

和众多其他数据库一样，Redis作为NoSQL数据库也同样提供了事务机制。在Redis中，MULTI/EXEC/DISCARD/WATCH这四个命令是我们实现事务的基石。相信对有关系数据库开发经验的开发者而言这一概念并不陌生，即便如此，我们还是会简要的列出Redis中事务的实现特征：

1. 在事务中的所有命令都将会被串行化的顺序执行，事务执行期间，Redis不会再为其他客户端的请求提供任何服务，从而保证了事务中的所有命令被原子的执行。
2. 和关系型数据库中的事务相比，在Redis事务中如果有某一条命令执行失败，其后的命令仍会被继续执行。
3. 我们可以通过MULTI命令开启一个事务，有关系型数据库开发经验的人可以将其理解为"BEGIN TRANSACTION"语句。在该语句之后执行的命令都将被视为事务之内的操作，最后我们可以通过执行EXEC/DISCARD命令来提交/回滚该事务内的所有操作。这两个Redis命令可被时为等同于关系型数据库中的COMMIT/ROLLBACK语句。
4. 在事务开启之前，如果客户端与服务器之间出现通讯故障并导致网络断开，其后所有待执行的语句都将不会被服务器执行。然后如果网络中断事件是发生在客户端执行EXEC命令之后，那么该事务中的所有命令都会被服务器执行。
5. 当使用Append-Only模式时，Redis会通过调用系统函数write将该事务内的所有写操作在本次调用中全部写入磁盘。然而如果在写入的过程中出现系统崩溃，如电源故障导致的宕机，那么此时也许只有部分数据被写入到磁盘，而另外一部分的数据却已经丢失。

Redis服务器会在重新启动时执行一系列必要的一致性检测，一旦发现类似问题，就会立即推出并给出相应的错误提示。此时我们就要充分利用Redis工具包中提供的redis-check-aof工具，该工具可以帮助我们定位到数据不一致的错误并将已经写入的部分数据进行回滚。修复之后我们就可以再次重新启动Redis服务器了。



## 13.WATCH命令和基于CAS的乐观锁

在Redis的事务中，WATCH命令可用于提供CAS(check-and-set)功能。假设我们通过WATCH命令在事务执行之前监控多了个keys，倘若在WATCH之后有任何key的值发生了变化，EXEC命令执行的事务都将被放弃，同时返回Null multi-bulk应答以通知调用者事务执行失败。例如，我们再次假设Redis中并未提供incr命令来完成键值的原子性递增，如果要实现该功能，我们只能字形编写相应的代码。其伪代码如下：

```haskell
val = GET mykey
val = val + 1
SET mykey $val
```

以上代码中哦有在单连接的情况下可以保证执行结果是正确的，因为如果在同一时刻有多个客户端在同时执行该段代码，那么就会出现多线程程序中经常出现的一种错误场景--竞态争用(race condition)。比如，客户端A和B都在同一时刻读取了mykey的原有值，结社该值为10，此后两个客户端又均将该值加一后set回Redis服务器，这样就会导致mykey的结果为11，而不是我们认为的12为了解决类似的问题，我们需要借助WATCH命令的帮助，见如下代码：

```haskell
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```

和此前代码不同的是，新代码在获取mykey的值之前先通过WATCH命令监控了该键，此后又将set命令包围在事务中美这样就可以有效的保证每个连接的执行EXEC之前，如果当前连接获取的mykey的值被其他连接的客户端修改，那么当前连接的EXEC命令将执行失败。这样调用者在判断返回值后就可以获悉val是否被重新设置成功。



## 14.Redis持久化的几种方式

1. 快照(snapshots)

   缺省情况下，Redis把数据快照存在磁盘上的二进制文件中，文件名为dump.rdb。

   你可以配置Redis的持久化策略，例如数据集中每N秒有超过M次更新，就将数据写入磁盘；或者你可以手工调用命令SAVE或BGSAVE。

   工作原理

   - redis forks
   - 子进程开始将数据写道临时RDB文件中。
   - 当子进程完成写RDB文件，用新文件替换老文件。
   - 这种方式可以使Redis使用copy-on-write技术。

2. AOF

   快照模式并不十分健壮，当系统停止，或者无意中将Redis kill掉，最后写入Redis的数据就会丢失。这对某些应用也许不是大问题，但对于要求高可靠性的应用来说，Redis就不是一个合适的选择。append-only文件模式是另一种选择。你可以在配置文件中打开AOF模式

3. 虚拟内存方式

   当你的key很小而value很大时，使用VM的效果会比较好，因为这样节约的内存比较大。

   当你的key不小时，可以考虑使用一些非常方法将很大的key变成很大的value，比如你可以考虑将key,value组合成一个新的value。

   vm-max-threads这个参数，可以设置访问swap文件的线程数，设置最好不要超过及其的核数，如果设置为0，那么所有对swap文件的操作都是串行的，可能会造成比较长时间 的延迟，但是对数据完整性有很好的保证。

   自己测试的时和发现用虚拟内存性能也不错。如果数据量很大，可以考虑分布时或者其他数据库



## 15. Redis的缓存失效策略和主键失效机制

作为缓存系统都要定期清理无效数据，就需要一个逐主键失效和淘汰策略。

在Redis当中，有生存期的key被成为volatile。在创建缓存时，要为给定的key设置生存期，当key过期的时候(生存期为0)，他可能会被删除。

1. 影响生存时间的一些操作

生存时间可以通过使用DEL命令来删除整个key来移除，或者被SET和GETSET命令覆盖原来的数据，也就是说，修改key对应的value和使用另外相同的key和value来覆盖以后，当前数据的生存时间不同。

比如说，对一个key执行INCR命令，对一个列表进行LPUSH命令，或者对一个哈希表执行HSET命令，这类操作都不会修改key本身的生存时间。另一方面，如果使用RENAME对一个key进行改名，那么改名后的key的生存时间和改名前一样。

RENAME命令的另一种可能是，尝试将一个带生存时间的key改名成另一个代生存时间的another_key，这时旧的another_key(以及它的生存时间)会被删除，然后旧的key会改名为another_key，因此，新的another_key的生存时间和也和原本的key一样。使用PERSIST命令可以在不删除key的情况下，移除key 的生存时间，让key重新成为一个persistent key。

2. 如何更新生存时间

可以对一个已经带有生存时间的key执行EXPIRE命令，新指定的生存时间会取代旧的生存时间。过期时间的精度已经被控制在1ms之内，逐渐失效的时间复杂度是O(1),

EXOIRE和TTL命令搭配使用，TTL可以查看key的当前生存时间。设置成功返回1；当key不存在或者不能为key设置生存时间时，返回0。

3. 最大缓存配置

在Redis中，允许用户设置最大使用内存大小 server.maxmemory 默认为0，没有指定最大缓存的情况下如果有新的数据添加超过最大内存，则会使Redis崩溃，所以一定要设置。Redis内存数据集大小上升到一定大小的时候们就会实行数据淘汰策略。

4. Redis提供的6种淘汰策略:

- volatile-lru: 从已设置过期时间的数据集(server.db[i].expires)中挑选最近最少使用的数据淘汰
- volatile-ttl: 从已设置过期时间的数据集(server.db[i].expires)中选择将要过期的数据淘汰
- volatile-random: 从已设置过期时间的数据集(server.db[i].expires)中任意选择数据淘汰
- allkeys-lru: 从数据集(server.db[i].dict)中挑选最近最少使用的数据淘汰
- allkeys-random: 从数据集(server.db[i].dict)中任意选择数据淘汰
- no-envictio(驱逐):禁止驱逐数据

注意这里的6种机制，volatile和allkeys规定了是对已设置过期时间的数据集淘汰数据还是从全部数据集淘汰数据，后面的lru、ttl以及random是三种不同的淘汰策略，再加上一种no-enviction永不回收的策略。

使用策略规则：

1. 如果数据呈现幂律分布，也就是一部分数据访问频率高，另一部分数据访问频率低，则使用allkeys-lru

2. 如果数据呈现平等分布，也就是所有的数据访问频率都相同，则使用allkeys-random

   三种数据淘汰策略：

   TTL和random比较容易理解，实现也会比较简单。主要时lru最近使用淘汰策略，设计上会对key按失效时间排序，然后取最先失效的key进行淘汰



## 16.Redis最适合的场景

Redis最适合所有数据in-memory的场景，虽然Redis也提供持久化功能，但实际更多的是一个disk-backed的功能，跟传统意义上的持久化有比较大的差别，那么大家可能就会有疑问，似乎Reids更像一个加强版的Memcached，那么何时使用Memcached，何时使用Redis呢？

如果简单地比较Redis和Memcached的区别，大多数都会得到以下观点：

1. Redis不仅仅支持简单的k/v类型的数据，同时还提供list、set、zset、hash等数据结构的存储。
2. Redis支持护具的备份，即master-slave模式的数据备份。
3. Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。



### 1.会话缓存(Session Cache)

最常用的一种使用Redis的情景时绘画缓存(Session Cache)。用Redis缓存会话比其他存储(如Memcached)的优势在于：Redis提供持久化。当维护一个不是严格要求一致性的缓存时，如果用户的购物车信息全部丢失，大部分人都会不高兴的，现在他们还会这样吗？
　　幸运的是，随着 Redis 这些年的改进，很容易找到怎么恰当的使用Redis来缓存会话的文档。甚至广为人知的商业平台Magento也提供Redis的插件。

### 2.全页缓存(FPC)

除基本的会话token之外，Redis还提供很简便的FPC平台。回到一致性问题，即使重启了Redis实例，因为有磁盘的持久化，用户也不会看到页面加载速度的下降，这是一个极大改进，类似PHP本地FPC。

再次以Magento为例，Magento提供一个插件来使用Redis作为全页缓存后端。

此外，对WordPress的用户来说，Pantheon有一个非常好用的插件 wp-redis ，这个插件能 帮助你以最快的速度加载你曾浏览过的页面。

### 3.队列

Redis在内存存储引擎领域的一大优点是提供 list 和 set  操作，这使得Redis能作为一个很好的消息队列平台来使用。Redis作为队列使用的操作，就类似于本地程序语言(如py)对 list 的 push/pop操作。

### 4.排行榜/计数器

Redis 内存中对数字进行递增或递减的操作实现的非常好。集合 (Set) 和有序集合 (Sorted Set) 也使得我们在执行这些操作的时候变得非常简单，Redis只是正好提供了这两种数据结构。

### 5.发布/订阅

最后（但肯定不是最不重要的）是Redis的发布/订阅功能。发布/订阅的使用场景确实非常多。我已看见人们在社交网络连接中使用，还可作为基于发布/订阅的脚本触发器，甚至用Redis的发布/订阅功能来建立聊天系统！（不，这是真的，你可以去核实）。
Redis提供的所有特性中，我感觉这个是喜欢的人最少的一个，虽然它为用户提供如果此多功能。



## 17.Redis实现分布式锁的思想

- 获取锁的时候，使用setnx加锁，并使用expire命令为锁添加一个超时时间，超过该时间则自动释放锁，锁的value值为一个随机生成的UUID，通过此在释放锁的时候进行判断。
- 获取锁的时候还设置一个获取的超时时间，若超过这个时间则放弃获取锁。
- 释放锁的时候，通过UUID判断是不是该锁，若是该锁，则执行delete进行锁释放。



## 18.Redis持久化

### 1.快照模式

或许在用Redis之初的时候，就听说过Redis有两种持久化模式，第一种是SNAPSHOTTING模式，还有一种是AOF模式，而且在实战场景下用过最多的莫过于SNAPSHOTTING模式，使用SNAPSHOTTING模式，需要在reids.conf中设置配置参数：

```redis

```

上面三组命令也是非常好理解的，就是说900指的是“秒数”，1指的是“change次数”，接下来如果在“900s“内有1次更改，那么就执行save保存，同样的道理，如果300s内有10次change，60s内有1w次change，那么也会执行save操作。



### 2.AOF

AOF持久化记录服务器执行的所有写操作命令，并在服务器启动时，通过重新执行这些命令来还原数据集。

AOF的默认策略为每秒钟fsync一次。（总是fsync 、从不fsync）

Redis还可以同时使用AOF持久化和RDB持久化。在这种情况下，当Redis重启时，它会优先使用AOF文件来还原数据集，因为**AOF文件保存的数据集通常比RDB文件所保存的数据集更完整**

父进程在保存RDB文件时唯一要做的就是fork出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无需执行任何磁盘I/O操作。RDB在挥发大数据集的速度比AOF的恢复速度要快。

AOF文件的体积通常要大于RDB文件。根据所使用的fsync策略，AOF的速度可能会慢于RDB。



## 19.Redis的并发竞争问题如何解决？

主要时发生在并发写竞争

1. 使用乐观锁的方式进行解决；(watch机制配合事务锁)
2. 排队的机制金牛星。讲所有需要对同一个key 的请求进行入队操作没然后用一个消费者线程从队里头依次独处请求并对相应的key进行操作。

这样对于同一个key的所有请求就都是顺序访问，正常逻辑下则不会有写失败的情况产生。从而最大化写逻辑的总体效率。

1. 客户端角度，为保证每个客户端之间正常有序与Redis进行通信面对连接进行池化，同时对客户端读写Redis操作采用内部锁synchronized。
2. 服务器角度，利用setnx实现锁。



## 20.Redis的缓存失效策略

影响生存时间的一些操作

**DEL命令，PERSIST**

如何更新生存时间

**带有生存时间的key执行EXPIRE命令，新指定的生存时间会取代旧的生存时间**

最大缓存配置






