**由哪些角色组成**

NameServer:名称服务器，保存Broker相关Topic等元信息并给生产者和消费者提供Broekr信息

Broker:消息存储中心，负责接收来自生产者的消息并存储，消费者从这里取得消息

Producer:生产者，负责生产消息

Consumer:消费者，负责从消息服务器拉去并消费消息



**NameServer**

nameServer用于存储Topic，Broker关系信息，功能较简单，稳定性高

1.多个NameServer之间互相没有通信，单个NameServer宕机不影响其他NameServer，

因为每个Broker向所有NameServer发送心跳，理想状态下，每个nameServer上的信息都一样

即使整个NameServer集群宕机，已经正常工作的其他角色仍能正常工作，但新启动的无法工作

2.主要负责维持心跳和提供Topic-Broker的关系数据。Broker向NameServer发送心跳时，会带上自己负责的所有Topic信息，

如果Topic过多，会导致一次心跳发送的数据过多，可能导致发送心跳失败，NameServer误以为Broker心跳失败



**Broker**

1.提供高并发读写服务，依靠一下两点

​	消息顺序写，所有Topic数据同时只会写一个文件，一个文件满1G,再写新文件，顺序写盘

​	消息随机读，Broker尽可能让读命中系统PageCache,操作系统访问PageCache时，即使只访问1K,的消息，也会预读出更多数据，下次读时就可能命中PageCache，减少IO操作

2.负载均衡与动态伸缩

​	负载均衡：Broker上存Topic信息，Topic由多个messageQueue组成，messageQueue平均分散在多个Broker上，Producer的发送机制保证消息尽可能平均发送到队列中。

​	动态伸缩：

​		Topic维度：如果一个Topic的消息量很大，但是集群压力不是很大，可以扩大Topic队列数量

​		Broker维度：如果集群压力比较大，需要扩容，直接家机器部署Broker即可，新机器启动后阿星NameServer注册，Producer、Consumer 通过 Namesrv 发现新Broker，立即跟该 Broker 直连，收发消息。

3.高可用

集群部署一般为主从，从节点实时从主节点同步消息，如果从节点宕机，从节点提供消息读服务

生产者发送消息是，Broker可以配置同步刷盘，消息写入磁盘才会返回成功，异步刷盘时，只有Broker宕机，才会丢失部分消息。

4.心跳机制

Broker跟所有NameServer保持心跳请求，每隔30s向所有NameServer发送心跳请求，包含Broker所有的Topic信息

NameServ会检查Broker的心跳信息，如果某个Broker2分钟没有发送心跳，则认为Broker下线。调整Topic和Broker的对应关系。

NameSErver不对主动通知其他角色，只有生产者，消费者下次拉取Topic信息的时候，才会发现有Broker宕机



**Producer**

1.拉取Topic-Broker的映射关系

​	Producer启动时，根据配置的NameServer地址，从集群中选一台建立长连接。如果这台NameServer宕机，会自动连接其他NameServer，知道找到一台可用的

​	生产者每30s从NameServer获取Topic和Broker的映射关系，更新到本地内存，然后跟所有Topic涉及的Broker建立长连接，每隔30s发送一次心跳

​	在Broker端也会每10s扫描一次当前注册的Producer，如果某个Producer超过2min没有发送心跳，则断连接

2.负载均衡

生产者发送消息时，轮询所有可发送的Broker，一条消息发送成功后，消磁换另一个Broker发送，达到消息平均分布的目的

几种发送方式：

同步，发送成功才返回

异步，回调接口通知发送结果

oneway，只发送，不管成不成功



**Consumer**

1.拉取Topic-Broker的映射关系

​	Consumer启动时，根据指定的NameServer地址，与其中一个 Namesrv 建立长连接。如果这台NameServer宕机，会自动连接其他NameServer，知道找到一台可用的

​	消费者每隔 30 秒从 Namesrv 获取所有Topic 的最新队列情况。连接建立后，从 Namesrv 中获取当前消费 Topic 所涉及的 Broker，直连 Broker 。消费者每30s发送心跳到Broker。在Broker端也会每10s扫描一次当前注册的Producer，如果某个Producer超过2min没有发送心跳，则断连接

2.负载均衡

不同的消费模式，有不同的负载均衡方法

集群消费：一个Topic可以由同一个消费这分组下的所有消费者分担消费。一个消息只会投递到某一个消费者那里，消费进度存储在Broker

广播消费:每个消费者消费Topic下的所有队列，一个消息只会投递到所有消费者那里，消费进度存储在消费者本地



**消费者获取消息的模式**





**顺序消息**

生产者顺序发送，消费者顺序消费



**定时消息**

消息发送到mq之后，不能立即被消费者消费，到特定的时间点或等待特定的时间才能被消费。目前只支持固定延迟级别的延迟消息，不支持任一时刻的消息



**消息重试**

消费者消息失败后，要提供重试机制，让消息再被消费一次

消费者消费失败后将失败的消息发送回Broker,进入延迟消息队列RETRY+TOPIC

1.消费者消费失败，将消息发送回Broker

2.Broker收到消息，置换Topic，存储消息

3.消费者会拉取该Topic对应的retryTopic下的消息

4.消费者拉取到重试消息后，置换到原始Topic，将消息消费

失败16次后，会存储到死信队列DLQ



**事务消息**



**高可用**

生产者：

1.生产者是发送消息的角色，无需考虑高可用

2.生产者配置多个NameServer列表，保证与NameServer连接的高可用，并定时拉取Topic信息

3.生产者和所有Broker直连，发送消息时，选择一个Broker发送，如果失败，会选用另一个Broker

4.生产者定时向Broker发送心跳，表明自己在线，Broker定时检测，判断是否有Producer下线



消费者：

1.消费者可配置多个节点，保证自身高可用。消费者分组中有新的消费者上线，或者有消费者下线，会重新分配Topic的Queue到存活的消费者

2.消费者配置多个NameServer,保证与NameServer连接的高可用,并定时拉取新的Topic信息

3.消费者和所有Broker直连，消费对应分配到的Queue的消息，如果消费事变，会发送回Broker

4.消费者定时向Broker发送心跳，表明自己在线，Broker定时检测，判断是否有Consumer下线



NameServer:

1.NameServer部署多个节点，保证自身高可用

2.NameServer不产生数据，Broker通过心跳将Topic信息同步到NameServer

3.NameServer之间灭有数据通信，Broker向多个NameServer发送心跳



Broker:

1.多个Broker形成一个分组，每个分组有一个Master节点和多个Slave节点。

​		Master可读可写，Slave可读

​		Master不断发送新的CommitLog到Slave节点，Slave节点不断上报本地CommitLog同步位置给Master

​		Slave节点从Master节点拉取消费进度，Topic配置等信息

2.多个Broker分组像喝茶呢个集群，集群之间没有通信

3.Broker可以配置同步刷盘，保证数据不丢失



**消息丢失 重复消费**

生产者同步发送消息，Broker同部刷盘可以不丢失数据

消费者宕机，未将消费进度发送到Broker可能导致重复消费，上线接口的幂等性可以防止重复消费



**那些情况用到消息队列**

消息队列的用处就是异步，解耦，削峰

异步，有些操作比较耗时，不能让用户一致等待，就返回一个中间状态，然后推送一条消息到消息队列中，在消费者里处理耗时的操作，tsa确权。

削峰，请求时有时候多有时候少的，使用消息队列就可以让请求均匀的消费。线索爬取消费

解耦，



高可用

nameServer多个实例，理论上每个实例里都有全部的路由信息，一个宕机不影响集群

broker集群，每个master有两个salve节点，master宕机了，salve晋升为master



重复消费问题

保证消费者接口的幂等性



不丢失

同步发送，同步刷盘



顺序消费

同一批消息进入同一个MessageQueue



