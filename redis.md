Redis是一种使用c语言开发的内存数据库

提供快速的读写服务，和多种数据类型。主要用作缓存，也可用于分布式锁消息队列。支持持久化



**几种数据结构**

### String:

#### 常用操作：

- **设置**：set key value [ex seconds] [px milliseconds] [nx|xx]
  ex:设置秒级过期时间;px:设置毫秒级过期时间;nx：不存在才可以设置成功，用于新增;xx：存在才可以设置成功，用于更新。
- **获取**：get key,不存在返回nil
- **批量设置**：mset key value [key value ...]
  例如：mset key1 value key2 value2
- **批量获取**：mget key [key ...]
  例如： mget key1 key2 key3
- **自增**： incr key,对可以对应的值自增
  如果值不是整数，返回错误;是整数返回自增的结果;不存在，按照值为0自增，返回1;
  类似的操作：decr(自减)、incrby(自增指定数字)、decrby(自减指定数字)、incrbyfloat(自增浮点数)
- **追加值**：append key value，向字符串尾部追加值
- **字符串长度**：strlen key，查看key对应value的长度，中文占三个字节
- **设置并返回原值**：getset key value，将key对应的值设为value，并返回之前的值
- **设置指定位置的字符**：setrange key offeset value，把key对应的值的下标为offset的那一位设置为value
- **获取部分字符串**：getrange key start end，前闭后闭
- **除了批量操作时间复杂度是操作的key的数量，其他操作时间复杂度都是O(1),获取部分字符串复杂度是O(n)**.

#### 内部编码：

**返回对象编码格式**： object encoding key
**redisObject**:

```
typedef struct redisObject {
        unsigned type:4;//类型
        unsigned encoding:4;//编码格式
        unsigned lru:REDIS_LRU_BITS; //实用LRU算法计算相对server.lruclock的LRU时间
        int refcount;//引用计数
        void *ptr;//指向底层数据实现的指针
    } robj;
```

- **int**: 8个字节的长整型：**对应底层实现long型整数**,保存的值为整型，如果更新为字符串类型，会自动转化为raw。

- **embstr**: 小于等于44个字节的字符串：embstr是一块连续的内存区域，只用分配一次内存，redisObject对象和sdshdr，值的内容只读，如果修改会自动转化为raw。

- raw

  : 大于44个字节的字符串：简单动态字符数组.

  - 简单动态字符串(sds)：使用动态字符串替换字符数组，将获取字符串长度所需的复杂度降低到O(1)，杜绝缓冲区溢出的情况；共有5中，分别为sdshdr5,8,16,32,64.
  - 除了sdshdr5之外，其它4个header的结构都包含3个字段:
  - len: 表示字符串的真正长度（不包含NULL结束符在内）
  - alloc: 表示字符串的最大容量（不包含最后多余的那个字节）
  - flags: 总是占用一个字节。其中的最低3个bit用来表示header的类型
  - 如果扩展后的字符串总长度小于1M则新字符串长度为扩展后的两倍；如果大于1M，则新的总长度为扩展后的总长度加上1M；这样做的目的是减少Redis内存分配的次数，同时尽量节省空间。
  - **惰性空间释放**: 惰性空间释放用于优化 SDS 的字符串缩短操作，当 SDS 的 API 需要缩短 SDS 保存的字符串时, 程序并不立即使用内存重分配来回收缩短后多出来的字节.

```
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; //实际长度
    uint8_t alloc; //总长度
    unsigned char flags; /.3bit表示header的类型
    char buf[];
};
```

### hash

#### 常用操作：

- **设置**: hset key field value,设置key对应的field的值为value.
- **获取**：hget key field,获取key为key对应的filed的值,如果键或field不存在，会返回nil
- **删除field**:hdel key field [field ...],可以删除一个或多个，返回结果为删除的个数
- **计算field个数**：hlen key 获取key对应的field的个数
- **批量设置field-value**: hmget key field [field ...]
- **批量获取field-value**: hmset key field value [field value ...]
- **判断field是否存在**：hexists key field
- **获取所有field**：hkeys key
- **获取所有value**：hvals key
- **获取所有的field-value**：hgetall key

#### 内部编码：

- **zipmap**:是一个连续的key value集合。 它的特点就是维护map需要的额外信息很少。最少可以到达只需要三个字节。不过这样做的后果就是时间复杂度为o(N),需要遍历才能操作。所以相对来说在定义上ZIPMAP_BIGLEN 就只有254，超过这个数量会转化为hashtable。
- **hashtable**:类似于Java中的hashmap。

```
typedef struct dictht {
    dictEntry **table;             // 哈希表数组
    unsigned long size;            // 哈希表数组的大小
    unsigned long sizemask;        // 用于映射位置的掩码，值永远等于(size-1)
    unsigned long used;            // 哈希表已有节点的数量
} dictht;
```

### list

#### 常用操作：

- **添加**：lpush key value [value ...],从左边添加；rpush key value [value ...],从右边添加;两种操作均可插入多个元素。
- **向某个元素前或者后插入元素**：linsert key before|after pivot value，在key对应的列表中，找到pivot在其前|后插入value。
- **获取指定范围内的元素列表**：lrange key start end，前闭后闭。从0到n-1；
- **获取列表指定索引下标的元素**: lindex key index,索引下标从左到右分别是0到N-1，但是从右到左分别是-1到-N。
- **获取列表长度**: llen key
- **删除**：lpop key,从左侧弹出元素；rpop key,从右侧弹出元素
- **删除指定元素**：lrem key count value，count>0，从左到右，删除最多count个元素; count<0，从右到左，删除最多count绝对值个元素; count=0，删除所有元素
- **按照索引范围修剪列表**：ltrim key start end，只留下start到end下标的元素。前闭后闭
- **修改指定索引下标的元素**：lset key index newValue

#### 内部编码：

- **quicklist**: 由ziplist组成的双向链表
- **ziplist**: 连续的内存块，由表头、若干个entry节点和压缩列表尾部标识符zlend组成，通过一系列编码规则，提高内存的利用率，使用于存储整数和短字符串。每次插入或删除一个元素时，都需要进行频繁的调用realloc()函数进行内存的扩展或减小，然后进行数据”搬移”，甚至可能引发连锁更新，造成严重效率的损失。是3.2之之前的list实现。
- **linkedlist**: 类似Java中的linkedlist

### set

#### 常用操作：

- **添加**：sadd key element [element ...],返回结果为添加成功的元素个数
- **删除元素**: srem key element [element ...],返回结果为成功删除元素个数
- **计算元素个数**: scard key
- **判断元素是否在集合中**: sismember key element
- **随机从集合返回指定个数元素**: srandmember key [count],count不写默认为1
- **从集合随机弹出元素**: spop key [count],同上
- **获取所有元素**：smembers key
- **求多个集合的交集**：sinter key [key ...]
- **求多个集合的并集**：suinon key [key ...]
- **求多个集合的差集**：sdiff key [key ...]

#### 内部编码：

- **intset**: 存储的是整型，且数量小于512.存储数据的时候是有序的.类似arraylist

```
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

- **hashtable**:同hash里的hashtable

### zset

#### 常用操作：

- **添加**：zadd key score member [score member ...]
- **计算成员个数**：zcard key
- **计算某个成员的分数**：zscore key member
- **计算成员的排名**：zrank key member;zrevrank key member
- **删除**: zrem key member [member ...]
- **增加成员的分数**: zincrby key increment member
- **返回指定排名范围的成员**: zrange key start end [withscores];zrevrange key start end [withscores]
- **返回指定分数范围的成员**: zrangebyscore key min max [withscores] [limit offset count];zrevrangebyscore key max min [withscores] [limit offset count]
- **返回指定分数范围成员个数**: zcount key min max
- **删除指定排名内的升序元素**: zremrangebyrank key start end
- **删除指定分数范围的成员**: zremrangebyscore key min max

#### 内部编码：

- **ziplist**:

- **skiplist**

  

**持久化**

将内存的数据转移到硬盘上的过程叫做持久化。防止应用或机器意外关闭丢失数据。

RDB:指定时间间隔对数据进行快照存储，可手动触发，自动触发。

手动触发save,不推荐使用，会阻塞Redis直到备份完成.bgsave，

自动触发，配置文件中 save second changes，xx秒内有xx次改动时触发bgsave。

AOF:每次对服务器的写操作时，追加记录到AOF文件中。



**bgsave**

流程：1.执行bgsave命令，redis进程判断是否有正在执行的子进程，如果有直接返回

2.redis进程fork一个子进程，fork操作会阻塞redis进程，

3.fork完成过后，bgsave返回background saving started,不再阻塞Redis进程

4.子进程创建rdb文件，根据父进程内存生成临时快照文件，替换原有文件

5.通知Redis进程备份完成



**aof**

配置文件中 appendonly yes，默认情况下未开启

文件名 appendfilename，默认appendonly.aof

流程：1.所有写命令追加到缓冲区中

2.缓冲区根据策略向硬盘做同步操作(每次操作，一秒一次(常用)，由操作系统控制)

3.定期对aof文件进行重写



重写：为了减少文件占用，重启时更快的加载

手动触发，bgrewriteaof

自动触发，auto-aof-rewrite-min-size,auto-aof-rewrite-percentage.到一定大小(默认64M),或者当前文件大小与上下次重写后aof文件的大小大于一定比值时自动触发。



**重启时加载数据**

1.开启了AOF,且存在AOF文件，优先加载AOF文件

2.不符合1，加载RDB文件



**两种备份方式对比**

RDB:保存了某个时间点的数据集，恢复更快，适用于灾难恢复，全量复制

发生意外丢失数据较多，数据大时，fork耗时



AOF:数据丢失少，配置每秒刷盘一次，最多丢失1秒数据

文件大小大于RDB,加载较慢



**线程模型**

文件时间处理器，io多路复用同时监听多个socket,根据socket上的时间选择对应的时间处理器处理。纯内存操作，非阻塞多路复用，避免多线程频繁上下文切换



**缓存穿透**

用户请求的数据在缓存中不存在，未命中缓存，且在数据库中也不存在，导致用户每次请求数据都请求数据库，然后返回空。

解决方法：

布隆过滤器：检验某个元素是否在一个集合中，一定不存在或可能存在。

返回空对象：缓存未命中，数据库查询为空，将空值写到缓存中，设置一个过期时间。

可能会浪费内存，短时间数据不一致



**缓存击穿**

热点key失效，大量请求数据库。

解决方法：

互斥锁：一个线程写缓存，其他线程等待缓存线程执行完，重新读取缓存，减低吞吐量

热点数据不过期：热点数据设置过期时间较长，且定时更新数据



**缓存雪崩**

一定时间内大批缓存过期，直接请求数据库

解决方法：

均匀过期:设置不同的缓存过期时间

互斥锁:同缓存击穿

永不过期:同缓存击穿



**内存淘汰机制**

配置文件 maxmemory  1024mb，默认为0，最大内存为系统剩余内存

淘汰策略：

noeviction:默认策略，写请求返回错误，不淘汰

allkeys-lru:从所有的key中使用LRU算法进行淘汰

volatile-lru:从设置了过期时间的key中使用LRU算法进行淘汰

allkeys-random:从所有key中随机淘汰

volatile-random:从设置了过期时间的key中随机淘汰

volatile-ttl:从设置了过期时间的key中，越早过期的优先被淘汰

allkeys-lfu:从所有可以中根据最少使用频率进行淘汰

volatile-lfu:从设置了过期时间的key根据最少使用频率进行淘汰

Volatile-X策略中，如果所有key都没设置过期时间，则返回错误

redis的lru维护一个大小为16的候选池，根据最近访问时间进行排序，第一次随机选择5个可以放入池中，

随后每次随机选择的key只有访问时间小于池中最小的时间的才会放到池中，直到放满，需要淘汰时，

从池中选择最近访问时间最小的key进行淘汰。



**过期删除策略**

**定时删除**：

在设置键的过期时间的同时，创建一个定时器(timer)，让定时器在键的过期时间来临时，立即执行对键的删除操作；对内存是最友好的：通过使用定时器，定时删除策略可以保证过期键会尽可能快地被删除，并释放过期键所占用的内存；但另一方面，定时删除策略的缺点是，他对CPU是最不友好的：在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，在内存不紧张但是CPU时间非常紧张的情况下，将CPU时间用在删除和当前任务无关的过期键上，无疑会对服务器的响应时间和吞吐量造成影响；

**惰性删除**：

放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，那就返回该键；对CPU时间来说是最友好的：程序只会在取出键时才对键进行过期检查，这可以保证删除过期键的操作只会在非做不可的情况下进行；惰性删除策略的缺点是，它对内存是最不友好的：如果一个键已经过期，而这个键又仍然保留在数据库中，那么只要这个过期键不被删除，它所占用的内存就不会释放；

**定期删除**：

每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于删除多少过期键，以及要检查多少个数据库，则由算法决定。是前两种策略的一种整合和折中： 定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响； 通过定期删除过期键，定期删除策略有效地减少了因为过期键而带来的内存浪费； 定期删除策略的难点是确定删除操作执行的时长和频率。



**主从复制**

将一台redis服务器的数据复制到其他redis服务器，前者为主节点，后者为从节点。复制是单向的，只能从主节点到从节点。

作用：

数据冗余:实现了数据热备份

故障恢复:主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复

负载均衡:可以设置读写分离，分担redis服务器负载，可以设置多从节点提高并发量

高可用:配合哨兵模式，集群模式可以实现redis高可用

存在的问题：

​	1.主节点故障，需要人工操作让从节点提供服务

​	2.写能力受制于主节点

​	3.存储能力受制于主节点

​	4.同步不成功会全量复制



**哨兵模式**

主从复制模式下，如果主节点故障，需要人工操作才能使从节点提供服务。

哨兵模式用来解决这一问题，能够自动完成故障发现与故障转移，并通知可用户端，实现高可用。

由一组哨兵节点(奇数个)和一组或多组主从复制节点组成。

1.每个哨兵节点每秒向其他哨兵节点和所有redis节点发送心跳检测。

2.如果一个redis节点距离最后一次回复心跳检测超过own-after-milliseconds，则认为这个节点主观下线。

3.如果一个Master被认为主观下线，则监测这个master的所有哨兵，以每秒一次的频率发送请求确认状态。

4.有足够多的哨兵(配置文件中配置，一般为哨兵总数/2+1)在指定时间范围内确定master处于主观下线状态，则master被标记为客观下线.

5.所有哨兵选举出一个哨兵，执行故障转移

6.选举出的哨兵，从从节点中选出一个作为主节点，

过滤不健康的节点，选择优先级最高的节点，选择复制偏移量最大的节点，选择runid最小的节点



**分布式锁**

1.set key value ex  nx

  可能会锁丢失，主节点未同步到从节点时，主节点宕机，从节点变为主节点，按时没有锁数据

2.RedLock

3.Zookeeper



**redis实现分布式限流**

zset实现，



**redis实现消息队列**

list作为队列，rpush生产消息，lpop消费消息。lpop没有消息，可以sleep一会重试

也可以blpop，在没有消息的时候阻塞，知道有消息

生产一次消费多次，pub/sub主题订阅者模式，在消费者下线的情况下，生产的消息会丢失



**redis实现延迟队列**

sortedset，内容作为key，时间戳作为score, zrangebyscore获取n秒之前的数据。



**redis事务**

事务中的命令串行执行，事务执行期间，redis不再为其他请求提供服务，保证了原子性

redis事务中的命令执行失败后， 后续命令会继续执行，不会回滚

multi开启事务

exec提交

discard回滚

 