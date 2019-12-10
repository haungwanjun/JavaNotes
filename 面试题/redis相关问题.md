1、项目中怎么使用redis的？

2、为什么Redis进行RDB持久化数据时，新起一个进程而不是在原进程中起一个线程来持久化数据

答案：主要是出于Redis性能的考虑，(1)Redis RDB持久化机制会阻塞主进程，这样主进程就无法响应客户端请求。(2)我们知道Redis对客户端响应请求的工作模型是单进程和单线程的，如果在主进程内启动一个线程，这样会造成对数据的竞争条件，为了避免使用锁降低性能。基于以上两点这就是为什么Redis通过启动一个进程来执行RDB了。

我们现在讨论redis的持久化机制，在Redis运行情况下，Redis 以数据结构的形式将数据维持在内存中，为了让这些数据在Redis 重启之后仍然可用，Redis 分别提供了RDB 和AOF 两种持久化模式。


首先我们来看一下Redis数据库在进行写操作时到底做了哪些事，主要有下面五个过程： 

客户端向服务端发送写操作（数据在客户端的内存中）。
数据库服务端接收到写请求的数据（数据在服务端的内存中）。
服务端调用write这个系统调用，将数据往磁盘上写（数据在系统内存的缓冲区中）。
操作系统将缓冲区中的数据转移到磁盘控制器上（数据在磁盘缓存中）。
磁盘控制器将数据写到磁盘的物理介质中（数据真正落到磁盘上）。
(一)RDB持久化机制

在Redis运行时，RDB程序将当前内存中的数据库快照保存到磁盘文件中，在Redis重启动时，RDB程序可以通过载入RDB文件来还原数据库的状态。RDB机制最主要的就是rdbSave和rdbLoad函数，前者将redis内存中数据加载到磁盘上，后者将在Redis重启时将数据恢复到redis内存中，注意rdbSave会阻塞主进程。

(1)RDB持久化机制启用方式：

Ⅰ：在Redis的配置文件中开启如下设置
   save 900 1     #900秒内如果超过1个key被修改，则发起快照保存
   save 300 10    #300秒内容如超过10个key被修改，则发起快照保存
   save 60 10000
Ⅱ:也可以通过手动执行SAVE和BGSAVE命令来执行保存快照到磁盘，SAVE和BGSAVE两个命令都会调用rdbSave函数,但它们调用的方式各有不同：
SAVE 直接调用rdbSave，阻塞Redis主进程看，直到保存完成为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
BGSAVE 则fork 出一个子进程，子进程负责调用rdbSave ，并在保存完成之后向主进程发送信号，通知保存已完成。因为rdbSave 在子进程被调用，所以Redis 服务器在BGSAVE 执行期间仍然可以继续处理客户端的请求。
如下的伪代码演示了SAVE和BGSAVE的大体逻辑：

```c++
def SAVE():
rdbSave()

def BGSAVE():
pid = fork()
if pid == 0:

子进程保存RDB

rdbSave()
elif pid > 0:

父进程继续处理请求，并等待子进程的完成信号

handle_request()
else:

pid == -1

处理fork 错误

handle_fork_error()
```

(2)RDB持久化机制优势
一旦采用该方式，那么你的整个Redis数据库将只包含一个文件，这样非常方便进行备份。比如你可能打算没1天归档一些数据。
方便备份，我们可以很容易的将一个一个RDB文件移动到其他的存储介质上
RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。
RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘操作。

(3)RDB持久化机制劣势
如果你需要尽量避免在服务器故障时丢失数据，那么 RDB 不适合你。 虽然 Redis 允许你设置不同的保存点（save point）来控制保存 RDB 文件的频率， 但是， 因为RDB 文件需要保存整个数据集的状态， 所以它并不是一个轻松的操作。 因此你可能会至少 5 分钟才保存一次 RDB 文件。 在这种情况下， 一旦发生故障停机， 你就可能会丢失好几分钟的数据。
每次保存 RDB 的时候，Redis 都要 fork() 出一个子进程，并由子进程来进行实际的持久化工作。 在数据集比较庞大时， fork() 可能会非常耗时，造成服务器在某某毫秒内停止处理客户端； 如果数据集非常巨大，并且 CPU 时间非常紧张的话，那么这种停止时间甚至可能会长达整整一秒。 虽然 AOF 重写也需要进行 fork() ，但无论 AOF 重写的执行间隔有多长，数据的耐久性都不会有任何损失。

(二)AOF持久化机制

AOF 则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到AOF文件，以此达到记录数据库状态的目的。AOF文件其实可以认为是Redis写操作的日志记录文件。

(1)AOF持久化机制的启用

在Redis的配置文件中开启如下设置：

```c++
appendonly yes              //启用aof持久化方式
appendfsync always      //每次收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用
appendfsync everysec     //每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐
appendfsync no    //完全依赖os，性能最好,持久化没保证
```

我们在文章的一开始就说了，Redis数据库在进行写操作时到底做了哪些事，主要有下面五个过程，其中有两步我们是要重点考虑的:

(WRITE)服务端调用write这个系统调用，将数据往磁盘上写（数据在系统内存的缓冲区中）。
这一步是每当有写操作执行时，Redis就会将数据写入到操作系统的缓存中

(SAVE)操作系统将缓冲区中的数据转移到磁盘控制器上（数据在磁盘缓存中）。
而这一步的发送时机就是我们在配置文件中配置的appendfsync的值有关

no值
在这种模式下写磁盘只会在以下任意一种情况中被执行： Redis 被关闭、AOF 功能被关闭、系统的写缓存被刷新（可能是缓存已经被写满，或者定期保存操作被执行） 这三种情况下的SAVE 操作都会引起Redis 主进程阻塞。

everysec值
在这种模式中，SAVE 原则上每隔一秒钟就会执行一次，因为SAVE 操作是由后台子线程调用的，所以它不会引起服务器主进程阻塞。

always值
在这种模式下，每次执行完一个命令之后，写磁盘操作都会被执行，同时主进程会被阻塞，不能接受客户端命令请求。

(2)AOF重写机制
aof 的方式也同时带来了另一个问题。持久化文件会变的越来越大。例如我们调用incr test命令100次，文件中必须保存全部的100条命令，其实有99条都是多余的。因为要恢复数据库的状态其实文件中保存一条set test 100就够了。

为了压缩aof的持久化文件。redis提供了bgrewriteaof命令。收到此命令redis将使用与快照类似的方式将内存中的数据 以命令的方式保存到临时文件中，最后替换原来的文件。

具体过程如下

redis调用fork ，现在有父子两个进程
子进程根据内存中的数据库快照，往临时文件中写入重建数据库状态的命令
父进程继续处理client请求，除了把写命令写入到原来的aof文件中。同时把收到的写命令缓存起来。这样就能保证如果子进程重写失败的话并不会出问题。

当子进程把快照内容写入已命令方式写到临时文件中后，子进程发信号通知父进程。然后父进程把缓存的写命令也写入到临时文件。
现在父进程可以使用临时文件替换老的aof文件，并重命名，后面收到的写命令也开始往新的aof文件中追加。

需要注意到是重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件,这点和快照有点类似。



(3)AOF持久化机制的优势
使用 AOF 持久化会让 Redis 变得非常耐久（much more durable）：你可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync 。 AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求）。



(4)AOF持久化机制的劣势
对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。
根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。



Redis有哪些数据结构？

字符串String、字典Hash、列表List、集合Set、有序集合SortedSet。

如果你是Redis中高级用户，还需要加上下面几种数据结构HyperLogLog、Geo、Pub/Sub。

如果你说还玩过Redis Module，像BloomFilter，RedisSearch，Redis-ML，面试官得眼睛就开始发亮了。

使用过Redis分布式锁么，它是什么回事？

先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。

这时候对方会告诉你说你回答得不错，然后接着问如果在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样？

这时候你要给予惊讶的反馈：唉，是喔，这个锁就永远得不到释放了。紧接着你需要抓一抓自己得脑袋，故作思考片刻，好像接下来的结果是你主动思考出来的，然后回答：我记得set指令有非常复杂的参数，这个应该是可以同时把setnx和expire合成一条指令来用的！对方这时会显露笑容，心里开始默念：摁，这小子还不错。

假如Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如果将它们全部找出来？

使用keys指令可以扫出指定模式的key列表。

对方接着追问：如果这个redis正在给线上的业务提供服务，那使用keys指令会有什么问题？

这个时候你要回答redis关键的一个特性：redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用scan指令，scan指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长。

使用过Redis做异步队列么，你是怎么用的？

一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。

如果对方追问可不可以不用sleep呢？list还有个指令叫blpop，在没有消息的时候，它会阻塞住直到消息到来。

如果对方追问能不能生产一次消费多次呢？使用pub/sub主题订阅者模式，可以实现1:N的消息队列。

如果对方追问pub/sub有什么缺点？在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如rabbitmq等。

如果对方追问redis如何实现延时队列？我估计现在你很想把面试官一棒打死如果你手上有一根棒球棍的话，怎么问的这么详细。但是你很克制，然后神态自若的回答道：使用sortedset，拿时间戳作为score，消息内容作为key调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。

到这里，面试官暗地里已经对你竖起了大拇指。但是他不知道的是此刻你却竖起了中指，在椅子背后。

如果有大量的key需要设置同一时间过期，一般需要注意什么？

如果大量的key过期时间设置的过于集中，到过期的那个时间点，redis可能会出现短暂的卡顿现象。一般需要在时间上加一个随机值，使得过期时间分散一些。

Redis如何做持久化的？

bgsave做镜像全量持久化，aof做增量持久化。因为bgsave会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要aof来配合使用。在redis实例重启时，会使用bgsave持久化文件重新构建内存，再使用aof重放近期的操作指令来实现完整恢复重启之前的状态。

对方追问那如果突然机器掉电会怎样？取决于aof日志sync属性的配置，如果不要求性能，在每条写指令时都sync一下磁盘，就不会丢失数据。但是在高性能的要求下每次都sync是不现实的，一般都使用定时sync，比如1s1次，这个时候最多就会丢失1s的数据。

对方追问bgsave的原理是什么？你给出两个词汇就可以了，fork和cow。fork是指redis通过创建子进程来进行bgsave操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来。

Pipeline有什么好处，为什么要用pipeline？

可以将多次IO往返的时间缩减为一次，前提是pipeline执行的指令之间没有因果相关性。使用redis-benchmark进行压测的时候可以发现影响redis的QPS峰值的一个重要因素是pipeline批次指令的数目。

Redis的同步机制了解么？

Redis可以使用主从同步，从从同步。第一次同步时，主节点做一次bgsave，并同时将后续修改操作记录到内存buffer，待完成后将rdb文件全量同步到复制节点，复制节点接受完成后将rdb镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。

是否使用过Redis集群，集群的原理是什么？

Redis Sentinal着眼于高可用，在master宕机时会自动将slave提升为master，继续提供服务。

Redis Cluster着眼于扩展性，在单个redis内存不足时，使用Cluster进行分片存储。





- RDB在保存RDB文件时父进程唯一需要做的就是fork出一个子进程,接下来的工作全部由子进程来做，父进程不需要再做其他IO操作，所以RDB持久化方式可以最大化redis的性能.
- 与AOF相比,在恢复大的数据集的时候，RDB方式会更快一些.

- **它保存了某个时间点得数据集,非常适用于数据集的备份**
- **会导致Redis在一些毫秒级内不能响应客户端的请求**

​    **AOF：.aof文件**

​        配置文件：

1. \# 是否开启AOF，默认关闭（no）  
2. appendonly yes  
3.   
4. \# 指定 AOF 文件名  
5. appendfilename appendonly.aof  
6.   
7. \# Redis支持三种不同的刷写模式：  
8. \# appendfsync always #每次收到写命令就立即强制写入磁盘，是最有保证的完全的持久化，但速度也是最慢的，一般不推荐使用。  
9. appendfsync everysec #每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，是受推荐的方式。  
10. \# appendfsync no 



- AOF文件是一个只进行追加的日志文件,所以不需要写入seek,即使由于某些原因(磁盘空间已满，写的过程中宕机等等)未执行完整的写入命令,你也也可使用redis-check-aof工具修复这些问题.
- Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写
- 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。
- AOF 文件的体积通常要大于 RDB 文件的体积



在运行情况下， Redis 以数据结构的形式将数据维持在内存中， 为了让这些数据在 Redis 重启之后仍然可用， Redis 分别提供了 RDB 和 AOF 两种[持久化](http://en.wikipedia.org/wiki/Persistence_(computer_science))模式。



在 Redis 运行时， RDB 程序将当前内存中的数据库快照保存到磁盘文件中， 在 Redis 重启动时， RDB 程序可以通过载入 RDB 文件来还原数据库的状态。

RDB 功能最核心的是 `rdbSave` 和 `rdbLoad` 两个函数， 前者用于生成 RDB 文件到磁盘， 而后者则用于将 RDB 文件中的数据重新载入到内存中：

![digraph persistent {    rankdir = LR;    node [shape = circle, style = filled];    edge [style = bold];    redis_object [label = "内存中的\n数据对象", fillcolor = "#A8E270"];    rdb [label = "磁盘中的\nRDB文件", fillcolor = "#95BBE3"];    redis_object -> rdb [label = "rdbSave"];    rdb -> redis_object [label = "rdbLoad"];}](http://redisbook.readthedocs.io/en/latest/_images/graphviz-cd96bfa5c61ef2b8dd69a9b0a97cde047cb722a8.svg)





`rdbSave` 函数负责将内存中的数据库数据以 RDB 格式保存到磁盘中， 如果 RDB 文件已存在， 那么新的 RDB 文件将替换已有的 RDB 文件。

在保存 RDB 文件期间， 主进程会被阻塞， 直到保存完成为止。

[SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 和 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 两个命令都会调用 `rdbSave` 函数，但它们调用的方式各有不同：

- [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 直接调用 `rdbSave` ，阻塞 Redis 主进程，直到保存完成为止。在主进程阻塞期间，服务器不能处理客户端的任何请求。
- [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 则 `fork` 出一个子进程，子进程负责调用 `rdbSave` ，并在保存完成之后向主进程发送信号，通知保存已完成。因为 `rdbSave` 在子进程被调用，所以 Redis 服务器在 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 执行期间仍然可以继续处理客户端的请求。

### 关于SAVE 、BGSAVE和BGREWRITEAOF执行区别：

- `rdbSave` 会将数据库数据保存到 RDB 文件，并在保存完成之前阻塞调用者。
- [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 命令直接调用 `rdbSave` ，阻塞 Redis 主进程； [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 用子进程调用 `rdbSave` ，主进程仍可继续处理命令请求。
- [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 执行期间， AOF 写入可以在后台线程进行， [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 可以在子进程进行，所以这三种操作可以同时进行。
- 为了避免产生竞争条件， [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 执行时， [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 命令不能执行。
- 为了避免性能问题， [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 和 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 不能同时执行。
- 调用 `rdbLoad` 函数载入 RDB 文件时，不能进行任何和数据库相关的操作，不过订阅与发布方面的命令可以正常执行，因为它们和数据库不相关联。发布与订阅功能和其他数据库功能是完全隔离的，前者不写入也不读取数据库，所以在服务器载入期间，订阅与发布功能仍然可以正常使用，而不必担心对载入数据的完整性产生影响。



### BGSAVE

在执行 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 命令之前， 服务器会检查 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 是否正在执行当中， 如果是的话， 服务器就不调用 `rdbSave` ， 而是向客户端返回一个出错信息， 告知在 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 执行期间， 不能执行 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 。

这样做可以避免 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 和 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 调用的两个 `rdbSave` 交叉执行， 造成竞争条件。

另一方面， 当 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 正在执行时， 调用新 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 命令的客户端会收到一个出错信息， 告知 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 已经在执行当中。

[BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 和 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 不能同时执行：

- 如果 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 正在执行，那么 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 的重写请求会被延迟到 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 执行完毕之后进行，执行 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 命令的客户端会收到请求被延迟的回复。
- 如果 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 正在执行，那么调用 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 的客户端将收到出错信息，表示这两个命令不能同时执行。

[BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 和 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 两个命令在操作方面并没有什么冲突的地方， 不能同时执行它们只是一个性能方面的考虑： 并发出两个子进程， 并且两个子进程都同时进行大量的磁盘写入操作， 这怎么想都不会是一个好主意。









下面特意写出来，自己看了原文后的困惑

原文：



### 

### SAVE

前面提到过， 当 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 执行时， Redis 服务器是阻塞的， 所以当 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 正在执行时， 新的 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 、 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 或 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 调用都不会产生任何作用。

只有在上一个 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 执行完毕、 Redis 重新开始接受请求之后， 新的 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 、 [BGSAVE](http://redis.readthedocs.org/en/latest/server/bgsave.html#bgsave) 或 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 命令才会被处理。

另外， 因为 AOF 写入由后台线程完成， 而 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 则由子进程完成， 所以在 [SAVE](http://redis.readthedocs.org/en/latest/server/save.html#save) 执行的过程中， AOF 写入和 [BGREWRITEAOF](http://redis.readthedocs.org/en/latest/server/bgrewriteaof.html#bgrewriteaof) 可以同时进行。



## 过期策略

定期删除，redis默认每隔100ms检查，是否有过期的key,有过期key则删除。需要说明的是，redis不是每隔100ms将所有的key检查一次，而是随机抽取进行检查(如果每隔100ms,全部key进行检查，redis岂不是卡死)。因此，如果只采用定期删除策略，会导致很多key到时间没有删除。

惰性删除，也就是说在你获取某个key的时候，redis会检查一下，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除。

过期策略存在的问题，由于redis定期删除是随机抽取检查，不可能扫描清除掉所有过期的key并删除，然后一些key由于未被请求，惰性删除也未触发。这样redis的内存占用会越来越高。此时就需要内存淘汰机制

内存淘汰机制
redis配置文件中可以使用maxmemory <bytes>将内存使用限制设置为指定的字节数。当达到内存限制时，Redis会根据选择的淘汰策略来删除键。（ps：没搞明白为什么不是百分比）

策略有如下几种：（LRU的意思是：Least Recently Used最近最少使用的，LFU的意思是：Least Frequently Used最不常用的）

volatile-lru -> Evict using approximated LRU among the keys with an expire set.
                    在带有过期时间的键中选择最近最少使用的。（推荐）
allkeys-lru -> Evict any key using approximated LRU.
                    在所有的键中选择最近最少使用的。（不区分是否携带过期时间）（一般推荐）
volatile-lfu -> Evict using approximated LFU among the keys with an expire set.
                    在带有过期时间的键中选择最不常用的。
allkeys-lfu -> Evict any key using approximated LFU.
                    在所有的键中选择最不常用的。（不区分是否携带过期时间）
volatile-random -> Remove a random key among the ones with an expire set.
                    在带有过期时间的键中随机选择。
allkeys-random -> Remove a random key, any key.
                    在所有的键中随机选择。
volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
                    在带有过期时间的键中选择过期时间最小的。
noeviction -> Don't evict anything, just return an error on write operations.

​                    不要删除任何东西，只是在写操作上返回一个错误。默认。