**sql语句执行过程**

sql语句执行，连接池链接，分析语法，查询缓存，优化器优化sql语句，执行器执行语句。



**mysql逻辑分层**

1.连接层，提供与客户端连接的服务

2.服务层，缓存查询，解析器，优化器。

3.引擎层，负责数据存储，存储引擎不同，存储方式，数据格式，提取方式也不同

4.存储层，存储数据，文件系统



**物理文件结构**

通用文件：所有存储引擎都有.frm文件保存每个表的元数据信息，存储表结构的定义信息

InnoDB:.idb文件，idbdata文件，idb文件是独享表空间存储方式，每个表一个.idb文件

.idbdate文件是共享表空间存储表格式，所有表共同使用一个或多个.idbdata文件

MyISAM:.MYD文件，存储数据   .MYI文件，存储索引



**InnoDB和MyISAM对比**

|        | MyISAM                     | InnoDB         |
| ------ | -------------------------- | -------------- |
| 外键   | 不支持                     | 支持           |
| 事务   | 不支持                     | 支持           |
| 行表锁 | 锁表，操作一条也会锁住整表 | 行锁，只锁一行 |
| 缓存   | 只缓存索引                 | 缓存索引和数据 |
| 表空间 | 小                         | 大             |
| 关注点 | 性能                       | 事务           |



**数据类型**

整数型：bit,tiny int,small int,medium int,int ,big int

浮点型：float,double,decimal

字符串型：char,varchar,tiny text,text,medium text,long text,tiny blob,blob,medium blob,long blob

日期类型：date,dateTime,timestamp,time,year



**三范式**

第一范式：列都是不可再分的，保证列的原子性，如果每列都是不可再分的最小数据单元，则满足第一范式

第二范式：满足第一范式，且表中非主键列不出存在对主键的依赖，要求每个表只描述一件事情

第三范式：满足第二范式，并且表中的列不存在对非主键列的传递依赖，非主键间不存在依赖关系。如订单表中存着客户编号符合第三范式，存着客户姓名则不符合第三范式



**事务隔离级别**

事务的四个属性：

原子性Atomicity：事务时一个完整的操作，全部成功或全部失败

一致性Consistency:事务完成时，数据必须处于一致状态,事务开始之前和结束之后，诗句哭的完整性约束没有被破坏

隔离性Isolation:对数据进行修改的所有并发事务彼此隔离，没有依赖关系

持久性Durability：事务完成后，对数据库的修改被永久保持。



隔离级别

读未提交READ-UNCOMMITTED：事务可以读到其他事务未提交的数据变更，可能导致脏读，不可重复度，幻读

读已提交READ-COMMITTED：事务执行过程中，可以读取到其他事务已提交的数据，可以防止脏读，会出现不可重复读，幻读

可重复读REPEATABLE-READ：对某一条数据的多次读取结果是一致的，除非数据被本事务修改，可防止脏读，不可重复读，会出现幻读。

序列化SERIALIZABLE:所有事务依次执行，可以防止以上三种

脏读：事务A读取了事务B修改的数据，然后B回滚了，A读到的是脏数据

不可重复度：事务A多次重复读取一条数据，事务B修改了这条数据，A两次读取到的数据不一致

幻读：事务A查询到N行数据，事务B插入一条。事务A又查到N+1条



**redolog,undolog,binlog的区别**

redolog,undolog存在于innodb中，统称为事务日志。

undolog用于回滚，事务未提交，改动没有全部生效，记录被部分修改，设备宕机，需要依靠undolog修复数据

redolog,事务提交后，部分数据写入磁盘，还有部分未写入，设备宕机，需要依靠redolog修复

binlog，归档日志，用于复制，主从同步，记录所有表数据结构变更和数据变更，不记录查询语句。



**多版本并发控制**

每条数据有两个隐藏字段，一个是trx_id,一个是roll_pointer.

trx_id是最近一次更新这个数据的事务ID，roll_pointer是执行这个事务执行之前生成的undo log

如事务A插入一条数据A,事务ID为10，则此时trx_id=10,roll_pointer执行空的undo log,事务B更新这条记录，

将值改为B,事务ID为15，此时trx_id=15，roll_pointer执行A,50，roll_pointer.

由此生成一条链。

ReadView:

事务开启时，新建一个ReadView,包含几个字段：

m_ids:生成时系统中活跃的事务的事务id列表

min_trx_id:生成时，系统中活跃的事务中id最小的

max_trx_id:生成时，系统应分配下一个事务id的大小

creator_trx_id:当前事务的id

某次读取数据，根据一下逻辑判断是否可见：

1.如果访问版本的trx_id与ReadView中的creator_trx_id相同，说明是本事务修改的记录，可以访问

2.如果trx_id小于ReadView中的min_trx_id，说明版本事务开启时trx_id事务已经提交，可以访问

3.如果trx_id大于或等于ReadView的max_trx_id,说明开启时，trx_id事务还没有提交，不能访问

4.如果trx_id在min_trx_id和max_trx_id之间时，如果trx_id在m_ids内，说明事务可能没有提交，不能访问

如果不在m_ids内，说明trx_id事务在ReadView生成之间就已经提交，，可以访问。

不能访问时，根据roll_pointer找到上一个版本的记录

读已提交这种隔离级别， 每个查询都会生成一个ReadView。可重复度只在事务开启时生成



**索引**

索引是一种帮助MYSQL快速获取数据的数据结构

索引一般数据结构为B+树，索引也是一张表，除主键索引外，其他索引保存了主键和索引字段，并指向实体表的记录。索引提高了查询的速度，降低了新增修改删除的速度，要维持索引的排序



B树与B+树的区别：

1.B+树非叶子结点不存具体数据，数据保存在叶子节点中

2.B+树所有叶子结点间有一个链指针，所以B+树适合随机访问和顺序访问，B树适合随机访问

3.B+树空间利用率高，节点的大小是固定的，非叶子结点不存储数据，可以存储更多的key，降低树的深度。



需要创建缩印的情况

1.主键自动创建唯一索引

2.频繁作为查询条件的字段

3.与其他表进行关联的字段

4.组合索引替代多个索引

5.查询中的排序字段

6.查询中统计或分组字段



不需要创建索引

1.数据太少

2.经常增删改的字段

3.数据重复且均匀的字段

4.频繁更新的字段



覆盖索引：

select的数据列从索引中就能够获得，不需要会标



**explain**

explain+SQL查看SQL执行计划

会出现几行数据

id:查询的序列编号，包含一组数组，表示查询中执行的顺序，

​	id大的先执行，id相同时，从上往下执行

select_type:查询类型

​	simple：简单的查询

​	primary:如果查询中包含复杂的子部分，最外层被标记为primay

​	subquery:在select或where列表中包含了子查询

​	derived:在from列表中包含的子查询被标记为derived，MySQL会递归的执行子查询，把	结果放到临时表中

​	union：union之后的查询语句，被标记为union，如果union包含在from子句的子查询	中，外层select被标记为derived

​	union result：从union表中获取的结果的select

table：这行数据是属于哪张表的

type：查询使用了那种类型，从好到差system->const->eq_ref->fulltext->ref_or_null->index_merge->unique_subquery->index_subquery->range->index->ALL

​	system:表只有一行记录

​	const：通过一次索引就找到了，获取主键列或唯一索引列

​	eq_ref:唯一索引扫描，对于每个索引建，表中只有一条数据匹配，常见于主键或唯一索	引扫描

​	ref:非唯一性索引扫描，范围匹配某一单独值的所有行，

​	range:只检索给定范围的行，使用一个索引来选择行，key列显示使用了哪个索引，范围	扫描索引好于全表扫描

​	index：索引全表扫描，遍历索引树，

​	ALL：数据全表扫描

possible_keys:可能使用到的索引列

key:实际使用到的索引

key_len:索引使用的长度

ref:索引的哪一列被使用了，如果可能的话是一个常数

rows:大致读取的行数

Extra:额外信息

​	using filesort:对数据使用外部的索引排序，无法利用索引完成的排序称为文件排序

​	using temporary:使用了临时表保存中间结果，常见于order by和group by

​	using index:使用了覆盖索引，效率不错；如果同时出现using where，表明索引用来执行索引键值的查找，否	则索引被用来读取数据而非执行查找操作

​	using where:使用了where过滤

​	using join buffer:使用了链接缓存



**SQL优化**

最左前缀法则，联合索引abc,可用的索引有a,ab,abc

索引上不进行计算，否则导致索引失效

尽量使用覆盖索引减少select

is null,is not null无法使用索引

like "xxx%"可以使用索引，like "%xx"无法使用索引

!= ,<> ,not in不能使用索引 

小表驱动大表:两个表筛选结果M,N。没有索引的情况下是一样的。驱动的结果集要进行

order by尽量使用索引进行排序



**删除大量数据**

1.删除索引

2.删除数据

2.新建索引



**limit 100000提高速度**

select  * from tbale where name = 'a' limit 1000000,100

--->select * form (

​	select id from table where name = 'a' limit 1000000,100

)a

Left join table b on a.id = b.id 

name要是索引列



**MySQL锁机制**

对数据的操作类型分类：

读锁和写锁：对同一份数据，多个读可以同时操作，写会阻断其他读和写

对数据操作的粒度分类：

表级锁：开销小，加锁快，不会出现死锁；锁粒度大，发生冲突概率大，并发度低，MyISAM和MEMORY存储引擎使用表级锁

行级锁：开销大，加锁慢；会出现死锁，锁粒度最小，冲突概率低，并发高，InnoDB支持行级锁和表锁，默认采用行级锁

页面锁：介于上面两者之间，会出现死锁。