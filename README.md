[![知识共享协议（CC协议）](https://img.shields.io/badge/License-Creative%20Commons-DC3D24.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh)

# 服务端技术

## redis

### 原理

- C 语言写成的，开源的高性能key-value非关系缓存数据库
- 内部使用文件事件处理器file event handler，这个处理器是单线程的，所以redis才叫做单线程模型

	- 多个socket
	- IO多路复用程序
	- 文件事件分派器
	- 事件处理器

- epoll模型

	- 用户态和内核态之间不用文件描述符（fd）的拷贝，而是通过mmap技术开辟共享空间，所有fd用红黑树存储，有返回结果的fd放在链表中，用户进程通过链表读取返回结果，伪异步I/O，性能较高

### 存储类型

- 基础结构

	- string

		- sds

	- list

		- quicklist

			- =ziplist+linkedlist

	- set

		- intset

			- 只能存整数
			- 针对整数有内存上的优化

		- dic

	- zset

		- 排序的set

			- ziplist

				- 元素数量小于128个
				- 所有member的长度都小于64字节

			- dict+skiplist

				- dic保存member到score
				- skiplist按照score排序存放member

	- hash

		- ziplist
		- dic

			- 结构也是数组加链表，基本同java的HashMap一致

- 高级结构

	- HyperLogLog

		- 以MurmurHash算法计算基数，记桶中当前的值为count

			- 低14位确定桶数组位置
			- 从15位开始数连续0（前导0），记为n，若n>count，存储n为count，否则不变

	- GEO

		- 底层是ZSet

			- member是具体业务对象
			- score是Geohash编码后的数据

	- BloomFilter

		- bitMap

			- 位图是⽀持按 bit 位来存储信息，用来实现布隆过滤器

				- 数量大了后会有误算率

### 发布/订阅

- channel

	- 底层是dic字典

		- key为channel
		- value为订阅的客户端

- pattern

	- 底层是一个链表

		- 每一个存储着具体pattern和订阅客户端

### 底层数据结构

- simple dynamic string(sds)

	- 结构

		- 记录字符串长度
		- 未使用的字节
		- 数据

	- 与c相比优点

		- 长度可以直接获得，c需要遍历
		- 内存分配时会多一点，修改时不一定重新分配内存，删除时也不马上释放内存
		- 存储二进制安全，c语言字符串中间不能包含'\0'（空）字符

- intset

	- 结构

		- encoding编码

			- 32位，标识存储的元素长度使用几位，如16，32，64位

		- length长度

			- 32位，表示存储元素个数

		- contents内容数据

			- 柔性数组，长度为encoding*length

- ziplist

	- 连续空间的数组

		- 减少内存碎片和指针占用的内存
		- 增加节点可能带来内存的复制移动

	- previous_entry_length(前一个节点的长度)

		- 本身的长度

			- 如果前一个节点小于254，占一个字节
			- 如果前一个节点不小于254，占五个字节

		- 增加和删除可能导致连锁更新

			- 每个节点都是252个字节，增加或者删除节点，导致某一个节点前的节点从小节点变成大节点，后面的都会更新

- quicklist

	- 使用ziplist和linkedlist组合，将linkedlist分段，每一段是ziplist，ziplist用双向链表的方式连接上一段和下一段ziplist

- skiplist

	- 多层结构

		- 插入是随机计算

	- 每一层是一个有序链表
	- 最底层包含所有元素
	- 如果N层有该元素，那么下面的层都有该元素
	- 每个节点两个指针，一个指向同一个链表的下一个元素，一个指向下面一层元素

- dict

	- 内部是两个hashtable

		- 结构也是数组加链表，基本同java的HashMap一致
		- 扩容时用到第二个

### 持久化方式

- rdb

	- 全量持久化

		- 如果持久化时有大量数据修改，性能会因为阻塞有所下降
		- save

			- 直接阻塞然后持久化

		- bgsave

			- fork子进程cow(copy on write)，复制触发时的内存更改，生成新的rdb文件

		- 默认配置

			- 900s 1
			- 300s 10
			- 60s 10000

- aof

	- 增量持久化

		- 记录操作日志
		- 配置3种模式appendfsync

			- always每一个操作就同步
			- everysec每一秒同步
			- no不主动同步

		- rewrite来保证过期数据的覆盖

### 内存淘汰机制

- no-eviction

	- 默认，内存不足时写入报错

- volatile

	- volatile-random

		- 从设置了过期时间的数据里随机挑选数据

	- volatile-ttl

		- 从设置了过期时间的数据里挑选要过期的数据

	- volatile-lru

		- 从设置了过期时间的数据里挑选最近最少使用的

	- 4.0后加入volatile-lfu

		- lfu算法

			- 淘汰率5%
			- 每个元素维护一个计数器，先随机选择样本大小的元素，然后按照使用频率排序后淘汰不常使用的

- allkeys

	- allkeys-random

		- 随机移除

	- allkeys-lru

		- 移除最近最少使用

			- lru算法

				- 淘汰率5%-5.5%
				- 每个元素维护一个时间戳，先随机选择样本大小的元素，然后按照时间排序后淘汰不常使用的

	- 4.0后加入allkeys-lfu

### 缓存雪崩

- 解释

	- 指一个时刻大量缓存失效，导致请求全部打到数据库

- 解决办法

	- 在过期时间中加入随机数，不让缓存同时失效

### 缓存穿透

- 解释

	- 通常是非法请求，比如构造负数的ID进行请求，缓存是没有的就会直接请求到mysql

- 解决办法

	- 参数校验
	- 使用波隆过滤器过滤
	- 不存在的数据设置短时间缓存

### 缓存与数据库数据一致性

- 延时双删

	- 过程

		- 先删缓存
		- 再写库
		- 根据业务时间延迟

			- 大于一次读请求到写缓存的时间

		- 再删缓存

	- 解决问题

		- 清除过程中可能存在的读导致的脏数据

	- 保证删除缓存

		- 引入mq
		- 引入binlog订阅程序和mq

### 集群

- maser slave主从

	- 配置

		- 使用slave of配置

	- 主从同步

		- 同步

			- 过程

				- slave向master发送sync
				- master执行bgsave，然后将生成的rdb和缓冲区的写命令发给slave

			- 场景

				- 初次同步
				- 断线后的同步

		- 命令传播

			- 过程

				- 只传播写命令(psync)

	- 特点

		- 异步从master节点复制数据
		- slave节点默认只读
		- master节点负责写

			- master宕机后重启会有一次重新同步，如果master节点数据没有持久化，会导致slave节点数据也丢失

- sentinel

  - 配置

```
# sentinel的固定配置格式sentinel <option_name> <master_name> <option_value>

# 配置sentinel监控的master
# sentinel监控的master的名字叫做mymaster，地址为127.0.0.1:6380
# sentinel在集群式时，需要多个sentinel互相沟通来确认某个master是否真的死了；
# 数字1代表，当集群中有1个sentinel认为master死了时，才能真正认为该master已经不可用了。
sentinel monitor mymaster 127.0.0.1 6380 1

# sentinel会向master发送心跳PING来确认master是否存活
# 如果master在“一定时间范围”内不回应PONG或者是回复了一个错误消息
# 那么这个sentinel会主观地认为这个master已经不可用了（SDOWN）
# 而这个down-after-milliseconds就是用来指定这个“一定时间范围”的，单位是毫秒。
sentinel down-after-milliseconds mymaster 5000

# 在发生failover主备切换时，这个选项指定了最多可以有多少个slave同时对新的master进行同步
# 这个数字越小，完成failover所需的时间就越长
# 但是如果这个数字越大，就意味着越多的slave因为replication而不可用。
# 可以通过将这个值设为 1 来保证每次只有一个slave处于不能处理命令请求的状态。
sentinel parallel-syncs mymaster 1

# 实现主从切换，完成故障转移的所需要的最大时间值。
# 若Sentinel进程在该配置值内未能完成故障转移的操作，则认为本次故障转移操作失败。
sentinel failover-timeout mymaster 60000

# 指定Sentinel进程检测到Master-Name所指定的“Master主服务器”的实例异常的时候，所要调用的报警脚本。
sentinel notification-script mymaster <script-path>
```

	- 原理
	
		- 哨兵
	
			- 功能
	
				- 集群监控：负责监控Redis master和slave进程是否正常工作
				- 消息通知：如果某个Redis实例有故障，那么哨兵可以通过API发送消息给管理员或者其他程序(配置里可见详情)
				- 故障转移：如果master node挂掉了，会自动转移到slave node上，并通知client客户端新的master地址
	
			- 工作方式
	
				- 定时发送PING，如果最后一次PING时间超过设置值，标记为主观宕机(SDOWN,subjective down)
				- 如果一个master被标记为SDOWN，那么监视这个master的所有sentinel都要以每秒一次去确认这个master的宕机情况，当大于等于设置数量的sentinel发现该master的SDOWN情况，就将该master标记为客观宕机(ODOWN,objectively down)
				- 尝试选举新的master,master的redis.conf、slave的redis.conf和sentinel.conf都会有所修改
	
			- 选举方式
	
				- 过滤故障的节点
				- 选择优先级slave-priority最大的从节点作为主节点，如不存在则继续
				- 选择复制偏移量（数据写入量的字节，记录写了多少数据。主服务器会把偏移量同步给从服务器，当主从的偏移量一致，则数据是完全同步）最大的从节点作为主节点，如不存在则继续
				- 选择runid（redis每次启动的时候生成随机的runid作为redis的标识）最小的从节点作为主节点
	
	- 特点
	
		- 高可用
		- 主从有的他都有

- cluster

	- 配置

```
# 另外两个端口分别是6381和6382（或者任意不重复即可）
port 6380
# 另外两个配置文件dir分别是/usr/local/var/db/redis_master2/、/usr/local/var/db/redis_master3/
dir /usr/local/var/db/redis_master1/
# 是否启用集群
cluster-enabled yes
# 另外两个配置文件分别为nodes-6381.conf、nodes-6382.conf
cluster-config-file nodes-6380.conf
# 配置集群节点的超时时间
cluster-node-timeout 15000
```

	- 启动
	
		- 连接节点
	
			- redis-cli -h 127.0.0.1 -p 6379
CLUSTER MEET 127.0.0.1 6380

		- 分配槽
	
			- redis-cli -h 127.0.0.1 -p 6380 cluster addslots {0..5461}
	
		- 配置slave
	
			- cluster replicate 7d480c106752e0ba4be3efaf6628bd7c8c124013（这个是6379节点的ID）
	
		- 重新分片
	
			- redis-cli --cluster reshard 127.0.0.1:6379
	
	- 原理
	
		- 通过虚拟16384个槽，然后根据节点分槽，提高性能
		- 节点之前直接通信
		- 连接任意一个节点即可
		- 节点的fail要超过半数的节点检测到
	
	- 特点
	
		- 扩展性

## 网络通信I/O模型

### BIO

- 过程

	- 主线程

		- 服务端等待客户端连接，阻塞进程
		- 检测到客户端连接，返回对应的文件描述符fd，并将连接移动到子线程
		- 开始下一次循环，跳回第1步

	- 子线程

		- 子线程中服务端等待客户端响应，阻塞进程
		- 接收到客户端响应，子线程处理
		- 开始下一次循环，跳回第1步

- 优势

	- 可以让每一个连接专注于自己的I/O并且编程模型简单，也不用过多考虑系统的过载、限流等问题

- 问题

	- 线程的创建和销毁成本很高
	- 线程本身占用较大内存
	- 线程的切换成本是很高的
	- 容易造成锯齿状的系统负载

### NIO

- 应用进程如果发起1K个请求，则在用户空间不停轮询这1K个socket文件描述符，查看是否有结果返回。这种方法虽然不阻塞，但是效率太低，有大量无效的循环
- 流程

	- 服务端请求客户端连接，非阻塞。如果没有直接返回null并向下执行
	- 如果有客户端连接，请求消息响应，非阻塞。如果没有响应直接返回null并继续遍历客户端
	- 遍历完所有客户端后，会重新开始循环，跳回第1步

- 优势

	- 规避多线程的问题
	- 单线程解决多任务

- 问题

	- 客户端循环遍历时，不断进行用户态和内核态的切换，系统调用开销非常大

### 多路复用I/O

- select

	- 能打开的文件描述符个数有限（最多1024个），如果有1K个请求，用户进程每次都要把1K个文件描述符发送给内核，内核在内部轮询后将可读描述符返回，用户进程再依次读取。因为文件描述符（fd）相关数据需要在用户态和内核态之间拷来拷去，所以性能还是比较低
	- 流程

		- 客户端建立连接后返回fd
		- 用二进制位表bitmap标记fds对应的位置，并将这个bitmap从用户态拷贝到内核态
		- 收到客户端响应，将对应的bitmap位标记为1
		- 程序处理消息，重新创建bitmap，循环下一次

	- 问题

		- bitmap默认最大限制1024位
		- bitmap不可重用，每次循环重新创建
		- 用户态和内核态切换开销
		- 轮询所有的客户端处理消息 O(n)

- poll

	- 可打开的文件描述符数量提高，因为用链表存储，但性能仍然不够，和selector一样数据需要在用户态和内核态之间拷来拷去
	- 流程

		- 与select相似，select中用bitmap标记fds，在poll中自己声明了结构体
		- 传输pollfd数组，解决了select中fds限制1024位的问题
		- 其次，内核将响应客户端对应的pollfd结构体中的revnets标记为1，说明这个客户端有消息响应
		- 处理消息时将这个pollfd的revnets重置即可，不用在重新创建数组，所以select中每次循环重新创建bitmap的问题也被解决了

	- 问题

		- 用户态和内核态切换开销
		- 轮询所有的客户端处理消息 O(n)

- epoll

	- 用户态和内核态之间不用文件描述符（fd）的拷贝，而是通过mmap技术开辟共享空间，所有fd用红黑树存储，有返回结果的fd放在链表中，用户进程通过链表读取返回结果，伪异步I/O，性能较高
	- 流程

		- 建立epoll对象时在内核分配资源，其数据结构是红黑树。添加和检索的时间复杂度 O(lgn)
		- 建立连接时，在红黑树中存储 epoll_event结构体，其中包含fd和events等信息
		- 调用epoll_wait收集响应的连接，放入一个单向链表

### AIO

- Linux下还是使用的epoll伪异步

## mysql

### 引擎

- InnoDB

	- 支持事务安全

- MyISAM

	- 插入数据快，空间和内存使用比较低。主要是用于插入新纪录和读出记录，对于并发性要求不高可以使用

		- 因为索引和数据分两个文件，根据索引直接到文件查找
		- 不需要关心事务

- MEMORY

	- 数据存储在内存中，可以使用缓存框架替代

### 事务特性ACID

- 一致性Consistency

	- 下面的三个特性都是为一致性服务

- 原子性Atomicity

	- 执行事务时生成undo.log

		- 失败时反向补偿

- 隔离性Isolation

	- 并发情况下，读操作可能存在的三类问题

		- 脏读

			- 直接读到其他事务的更新

		- 不可重复读

			- 两次情况下，读到了其他事务的更新

		- 幻读

			- 两次情况下，读到了其他事务的新增和删除

	- 事务隔离等级

		- Read Uncommitted读未提交

			- 会出现脏读、不可重复读、幻读
			- 写时加入行级共享锁

		- Read Committed读已提交

			- 解决脏读

				- 快照读
				- 写时加入行级排他锁

			- 会出现不可重复读、幻读

		- Repeatable Read可重复读(InnoDB默认)

			- 解决脏读和不可重复读

				- 解决方案

					- Multi-Version Concurrency Control，即多版本的并发控制协议

						- 隐藏列

							- 包含了数据的事务ID、指向undo.log的指针

						- undo.log的版本链

							- 每条undo.log会指向更早的undo.log

						- ReadView

							- 指事务（记做事务A）在某一时刻给整个事务系统（trx_sys）打快照，之后再进行读操作时，会将读取到的数据中的事务id与trx_sys快照比较，从而判断数据对该ReadView是否可见，即对事务A是否可见

								- trx_sys

									- low_limit_id：表示生成ReadView时系统中应该分配给下一个事务的id。如果数据的事务id大于等于low_limit_id，则对该ReadView不可见
									- up_limit_id：表示生成ReadView时当前系统中活跃的读写事务中最小的事务id。如果数据的事务id小于up_limit_id，则对该ReadView可见
									- rw_trx_ids：表示生成ReadView时当前系统中活跃的读写事务的事务id列表。如果数据的事务id在low_limit_id和up_limit_id之间，则需要判断事务id是否在rw_trx_ids中：如果在，说明生成ReadView时事务仍在活跃中，因此数据对ReadView不可见；如果不在，说明生成ReadView时事务已经提交了，因此数据对ReadView可见

			- 会出现幻读

				- 部分解决方案

					- next key lock间隙锁

				- 查询手动加入锁可以解决

		- Serializable可串行化

			- 安全

				- 加入表级共享锁和排他锁

	- 锁类型

		- 行锁

			- row lock记录锁

				- 主键查询情况下，锁住主键索引
				- 非主键索引情况下，锁住非主键索引再锁对应主键索引

			- next key lock临键锁

				- 记录锁+gap lock间隙锁

					- 根据索引查询条件，先分析存在哪些间隙，左开右闭；在非主键索引相同的情况下会按照主键排序

		- 表锁

			- 如果没有明确用到索引，就是索引失效的情况下要全标扫描就会使用表锁

	- 锁模式

		- 意向共享锁IS
		- 意向排他锁IX
		- 共享锁S
		- 排他锁X
		- 自增锁AUTO_INC

- 持久性Durability

	- 执行事务时生成redo.log
	- 因为在执行操作时，先更新buffer pool，在更新到磁盘

### 主从同步

- 过程

	- 主节点

		- binlog dump thread

			- 负责将更新时间存入binlog，然后由log dump线程通知从节点有更新

	- 从节点

		- I/O thread

			- 该线程用于请求Master，Master会返回binlog的名称以及当前数据更新的位置、binlog文件位置的副本。
然后，将binlog保存在 「relay log（中继日志）」 中，中继日志也是记录数据更新的信息

		- SQL thread

			- 检测中继日志的更新，将内容更新到从节点

- 策略

	- 同步策略

		- 确认所有从节点收到事务主节点再提交

	- 半同步策略

		- 有一个从节点收到事务主节点再提交

	- 异步策略

		- 只通知从节点同步，然后自己直接提交，不等待从节点是否收到事务

### 优化

- 索引优化

	- 基础

		- 索引类型

			- Normal普通索引
			- FULLTEXT全文索引
			- UNIQUE唯一索引
			- SPATIAL空间索引

				- 暂时不考虑

		- 聚簇索引 & 非聚簇索引

			- 聚簇索引

				- 主键索引

					- 叶子节点是索引以及数据

				- 二级索引（非聚簇索引）

					- 叶子节点是索引以及主键，通过主键在找到数据

						- 引入自适应哈希

							- 经常访问的二级索引数据会自动被生成到hash索引里面去(最近连续被访问三次的数据)
							- 占用buffer pool
							- 等值查找才有效

			- 非聚簇索引

				- 主键索引和二级索引一样，叶子节点存放的是索引和索引对应记录的指针

		- 索引方法

			- BTREE

				- 用B+Tree和顺序链表实现

					- B+Tree，每一个节点是一个[key,data]的二元数组，非叶子节点不存储data，只存储索引key；只有叶子节点才存储data
					- mysql在B+Tree的基础上增加了叶子节点到相邻叶子节点的指针

			- HASH

				- 不支持范围索引，等值、不等值的过滤很快

	- 索引失效

		- 违反最左匹配法则

			- 例如idx_col1_col2_col3，where后查询的是col2=? and col3=?

		- 在索引列上做操作
		- 索引范围右边的列会失效
		- 使用!=，<>会导致全表扫描
		- like以通配符开头('%abc')
		- 字符串不加单引号
		- 使用or
		- order by非索引字段或者order by违反最左匹配原则
		- group by非索引字段或者group by违反最左匹配原则

	- 索引覆盖

		- 在二级索引中，如果 where 条件的列和 select 的列都在一个索引中，通过这个索引就可以完成查询，不用再回表

	- 索引下推ICP

		- 假如有一个table(a,b,c,d,e)，a是主键，有一个index_b_c_d(b,c,d)，那么在做下面操作的时候会有一个索引下推
		- select * from table where b>2 and b<7 and c>0 and d!=0 and e!=0;
		- 首先在b的范围搜索后的列索引失效
		- 在没有索引下推的流程里

			- server层

				- index_filter

					- c>=,d!=0

				- table_filter

					- e!=0

			- 引擎层

				- index_key

					- index_first_key

						- a>2,c>0

					- index_last_key

						- a<7

		- 在有索引下推的流程里(将index_filter下推)

			- server层

				- table_filter

					- e!=0

			- 引擎层

				- index_key

					- index_first_key

						- a>2,c>0

					- index_last_key

						- a<7

				- index_filter

					- c>=,d!=0

		- 总结

			- 下推指是在二级索引里的index_filter下推，可以在二级索引里直接过滤index_filter，减少需要回表查询的行数

	- expalin

		- type

			- all

				- 全表扫描

			- index

				- 索引扫描

			- range

				- 只索引给定范围的行，例如范围查询使用到了索引

			- ref

				- 连接匹配条件，即哪些列或者常量用于查找索引列上的值

			- eq_ref

				- 和上面一样，不过使用的是唯一索引

			- const

				- 对查询的优化，转化为一个常量，例如where id=

			- NULL

				- 优化到不需要使用表或者索引，例如直接从一个索引列通过单独索引查找

		- possible_keys

			- 可能用到的索引

		- key

			- 使用的索引

		- extra

			- using filesort

				- 无法使用索引排序

			- using index

				- 使用索引，内容可以从索引中拿到

			- using where

				- 使用了where过滤条件

			- using temporary

				- 使用了临时表，常见与排序和分组查询

			- using join buffer

				- 连接时没有用到索引，需要缓冲区缓冲中间结果

			- impossible where

				- where可能没有复合条件的行

			- select tables optimized away

				- 仅通过索引，优化器可能仅从聚合函数结果中返回一行

- 大数据插入优化

	- mysqldump加入skip-extended-insert以一行数据的形式导出再插入
	- 提交时手动关闭自动事务，然后手动加入事务，减少事务创建
	- 去掉其他索引，插入后再创建索引
	- 修改为MyISAM插入后再改回来

- 死锁排查

	- show engine innodb status查看死锁日志

		- 结合binlog查看事务具体执行的sql

			- 结合代码分析

- like %开头不走索引优化

	- 使用反转函数，走反转索引

- 多表联查优化

	- 充分利用索引
	- 可以利用子表自己先筛选再连接
	- 使用union手动替代临时表后再排序

- 上亿数据如何进行分页查询

	- 

		- 顺序访问

			- 充分利用上一页的偏移量来进行

		- 随机访问

			- 单一索引

				- 利用聚簇索引查询获取id，再用ID进行查询

			- 复合索引

				- 充分满足索引优化去做，从where到order by的复合索引建立

### 分库分表

- 描述

	- 水平拆分

		- 减小索引速度

	- 垂直拆分

		- 业务设计时的问题

- 实现

	- 简单实现

		- 根据主键手动取模

	- shardingJDBC

		- 描述

			- 无需多余部署
			- 与业务耦合较高

		- 构成

			- 逻辑表
			- 真实表
			- 数据节点
			- 分片算法

				-  1.精确分片算法（PreciseShardingAlgorithm）：用于处理使用单一键作为分片键的=和IN的场景,需要配合StandardShardingStrategy使用
				-  2.范围分片算法（RangeShardingAlgorithm）：用于处理使用单一键作为分片键的between、and、>、<、>=、<=等分片场景,需要配合StandardShardingStrategy使用
				-  3.复合分片算法（ComplexKeysShardingAlgorithm）：用于处理使用多键作为分片键进行分片的场景，包含多个分片键的逻辑比较复杂，需要配合ComplexShardingStrategy使用
				-  4.Hint分片算法（HintShardingAlgorithm）：用于处理Hint行分片的场景，需要配合HintShardingStrategy使用

			- 分片策略

				- 1.标准分片策略：对应StandardShardingStrategy,提高对SQL语句中的=,>,<,>=,<=,IN,AND,BETWEEN等分片操作的支持.StandardShardingStrategy只支持单键分片,提供PreciseShardingAlgorithm和RangeShardingAlgorithm两个分片算法。PreciseShardingAlgorithm是必选的，用于处理=和IN的分片。RangeShardingAlgorithm是可选的，用于处理BETWEEN AND, >, <, >=, <=分片，如果不配置RangeShardingAlgorithm，SQL中的BETWEEN AND将按照全库路由处理
				-  2.复合分片策略：对应ComplexShardingStrategy。复合分片策略。提供对SQL语句中的=, >, <, >=, <=, IN和BETWEEN AND的分片操作支持。ComplexShardingStrategy支持多分片键，由于多分片键之间的关系复杂，因此并未进行过多的封装，而是直接将分片键值组合以及分片操作符透传至分片算法，完全由应用开发者实现，提供最大的灵活度
				-  3.行表达式分片策略：对应InlineShardingStrategy。使用Groovy的表达式，提供对SQL语句中的=和IN的分片操作支持，只支持单分片键。对于简单的分片算法，可以通过简单的配置使用，从而避免繁琐的Java代码开发，如: t_user_$->{u_id % 8} 表示t_user表根据u_id模8，而分成8张表，表名称为t_user_0到t_user_7。
				-  4.Hint分片策略：对应HintShardingStrategy,通过Hint指定分片值而非从SQL中提取分片值的方式进行分片的策略
				-  5.不分片策略：对应NoneShardingStrategy,不分片的策略

		- 分布式ID

			- UUID
			- Redis incr()
			- 雪花算法

				- 构成

					- 时间ID

						- 41bit

					- 机器ID

						- 10bit

					- 序列ID

						- 12bit

				- 性能

					- 一个节点每毫秒4096个ID序号
					- 服务最大每毫秒409.6万个序列号

				- 时钟问题

					- 服务器时钟回拨会导致产生重复序列,因此默认分布式主键生成器提供了一个最大容忍的时钟回拨毫秒数。如果时钟回拨的时间超过最大容忍的毫秒数值,则程序报错;如果在可容忍的范围内,默认分布式主键生成器会等待（Thread.sleep）时钟同步到最后一次主键生成的时间后再继续工作。最大咨忍的时钟回拨室秒数的默认值为0,可通过调用静态方法 Defaultkey Generator setMaxTolerate Time DifferenceMilliseconds设置

		- 分布式事务

	- mycat

		- 描述

			- 单独部署服务
			- 业务没有感知

- 首次分库

	- 双写迁移

- 动态扩容缩容

	- 缩容扩容考虑为2的次幂

## mybatis

### 缓存机制

- 一级缓存

	- localcache
	- 作用域

		- session
		- statement

- 二级缓存

	- enableCache
	- 作用域

		- namespace

			- 多个sqlSession共享
			- 即Mapper级别

- 建议

	- 默认的一二级缓存实现都是基于本地的，所以在分布式系统下会出现脏数据，不能直接使用，需要再开发。考虑redis替代

### #{}和${}的区别

- #{}

	- 说明

		- 会在预编译中解析为一个占位符，变量的替换在数据库中完成

	- 优点

		- 相同的预编译 sql 可以重复利用

- ${}

	- 在预编译前的SQL动态解析的标量替换
	- 可能会有SQL注入的问题

### SQL动态解析

- 占位符处理
- 动态SQL处理
- 参数类型校验

### 拦截器

- 实现org.apache.ibatis.plugin.Interceptor下的三个方法

	- intercept

		- 具体操作

	- plugin

		- 加入拦截链

	- setProperties

		- 自定义设置属性

- 使用注解

	- 加密

		- @Intercepts({   @Signature(type = ParameterHandler.class, method = "setParameters", args = PreparedStatement.class)
})

	- 解密

		- @Intercepts({   @Signature(type = ResulSetHandler.class, method = "handleResultSet", args = Statement.class)
})

	- 参数

		- type

			- 指定当前拦截器使用StatementHandler 、ResultSetHandler、ParameterHandler，Executor的一种

		- method

			- 使用以上四种类型的具体方法

		- args

			- 指定预编译语句

### mybatis-plus

- 本地多数据源事务

	- threadLocal的value是一个concurrentHashmap，存放一个线程中的所有数据源，最后进行统一提交和回滚

## mq

### rabbitmq

- 特点

	- erLang开发，天然支持分布式
	- 延时任务

		- 通过TTL和死信队列

- 结构

	- VHost

		- 虚拟Broker，mini-Rabbitmq server

	- Broker(一个rabbitmq server)

		- Exchange
		- RoutingKey
		- Queue

	- Producer
	- Consumer

- 持久化

	- 特点

		- 顺序写，随机读
		- 只能少量堆积
		- 不能保证数据不丢失

	- 触发条件

		- buffer满了
		- 周期性持久化，25ms

	- 配置

		- exchange持久化

			- 设置durable 为 true

		- queue持久化

			- 设置durable 为 true

		- message持久化

			- 设置delivery_mode为2

- 工作模式

	- simple

		- 生产者直接发消息到queue，一个消费者直接从队列里面消费，没有exchange

	- work

		- 生产者直接发消息到queue，多个消费者直接从队列里面消费，没有exchange

	- routing

		- exchange通过routingkey将消息发给指定队列，模式direct

	- publish/subscribe

		- exchange将消息发给下面的所有队列，模式fanout

	- topic

		- exchange通过正则routingkey将消息分发给多个队列，模式topic，routing key使用通配符

- 消息丢失

	- 生产者到server

		- 开启confirm模式

	- server持久化

		- 镜像集群

	- server到消费者

		- 开启手动ack

- 消费堆积

	- 排查消费者的消费性能瓶颈
	- 增加消费者的多线程处理
	- 部署增加多个消费者

- 死信队列

	- 进入情况

		- 消息被否定，使用了channel.basicNack或者channel,basicReject，且requeue为false
		- 超过TTL时间
		- 队列的消息数量超过了最大队列长度

- 集群

	- 普通模式

		- 仅作元数据的同步

			- vhost信息、user信息
			- exchange信息
			- queue信息
			- exchange和queue的绑定信息

	- 镜像模式

		- slave只做备份和主备切换
		- master和多个slave会构成一个环，最老的slave会成为下一个master
		- 可以通过平均分配master节点，形成负载均衡

### rocketmq

- 特点

	- Java开发
	- 分布式
	- 单队列百万级消息容量

- 结构

	- NameServer

		- 特点

			- 主要负责对于源数据的管理，包括了对于Topic和路由信息的管理
			- 每个NameServer节点互相之间是独⽴的，没有任何信息交互

	- Broker

		- Topic

			- 消息的以及类型，⼀条消息必须有⼀个主题

		- Tag

			- 消息的二级类型

		- group

			- 一个组可以订阅多个topic，用于设置producer或者consumer到一个组里，横向扩展

		- queue

			- 一个topic下可以有多个queue，用于集群模式下的扩展

	- producer
	- Consumer

- 持久化

	- 原理

		- CommitLog

			- 消息真正的存储文件，所有消息都顺序存储在 CommitLog 文件中

		- IndexFile

			- 消息索引文件，主要存储消息 Key 与 offset 对应关系，提升消息检索速度

		- ConsumeQueue

			- 消息消费逻辑队列，类似数据库的索引文件

	- 消息数据持久化

		- 原理

			- pageCache，利用OS空闲的内存作为磁盘缓存

		- 方式

			- 同步刷盘

				- 消息存入后同步刷入磁盘，数据安全，吞吐量不大

			- 异步刷盘

				- 消息存入缓存，到达一定量后再同步到磁盘，吞吐量大，数据不安全

- 分布式事务

	- half消息

		- 生产者向MQ服务器发送half消息
		- half消息发送成功后，MQ服务器返回确认消息给生产者
		- 根据本地事务执行的结果（UNKNOW、commit、rollback）向MQ Server发送提交或回滚消息
		- 如果错过了（可能因为网络异常、生产者突然宕机等导致的异常情况）提交/回滚消息，则MQ服务器将向同一组中的每个生产者发送回查消息以获取事务状态
		- 生产者根据本地事务状态发送提交/回滚消息
		- MQ服务器将丢弃回滚的消息，但已提交（进行过二次确认的half消息）的消息将投递给消费者进行消费

- 消费模式

	- 集群消费

		- 一条消息只会被同Group中的一个Consumer消费
		- 多个Group同时消费一个Topic时，每个Group都会有一个Consumer消费到数据

	- 广播消费

		- 消息将对一个Consumer Group 下的各个 Consumer 实例都消费一遍

- 延迟任务

	- 设置DelayTimeLevel，每个level的具体值可以在配置文件里配置

- 集群

	- 单master
	- 多master

		- 配置文件2m-nosalve

	- 多master多slave

		- 异步复制

			- 配置文件2m-2s-async

		- 同步双写

			- 配置文件2m-2s-sync

## zookeeper

### 基本特征

- 通知机制
- 精简的文件系统

### 集群同步

- Zab协议

	- 恢复模式

		- leader宕机或者没有过半的follower与leader保持通信，触发恢复，选举新的leader，并进行数据同步

	- 广播模式

		- 完成数据同步后，进入广播模式，处理请求

- 事务一致性

	- 事务ID zxid

		- 高32位表示该事务发生的集群选举周期,集群每发生一次leader选举，值加1
		- 低32位为递增计数,leader每处理一个事务请求，值加1，发生一次leader选择，低32位要清0

### 角色

- 三种

	- Leader

		- 事务请求的唯一调度、处理者
		- 与follower进行数据同步

	- Follower

		- 处理非事务请求，转发事务请求给leader
		- 参与leader选举投票

	- Observer

		- 和follower一样，除去参与leader选举投票

- 状态

	- Leading

		- 被选中的leader

	- Following

		- 与选出的leader同步

	- Observing

		- 行为和following基本一致，只是不选举，不投票

	- Looking

		- 寻找Leader中

- 选举算法

	- basic paxos

		- 选举线程由当前Server发起选举的线程担任，其主要功能是对投票结果进行统计，并选出推荐的Server
		- 选举线程首先向所有Server发起一次询问(包括自己)
		- 选举线程收到回复后，验证是否是自己发起的询问(验证zxid是否一致)，然后获取对方的id(myid)，并存储到当前询问对象列表中，最后获取对方提议的leader相关信息(id,zxid)，并将这些信息存储到当次选举的投票记录表中
		- 收到所有Server回复以后，就计算出zxid最大的那个Server，并将这个Server相关信息设置成下一次要投票的Server
		- 线程将当前zxid最大的Server设置为当前Server要推荐的Leader，如果此时获胜的Server获得n/2 + 1的Server票数， 设置当前推荐的leader为获胜的Server，将根据获胜的Server相关信息设置自己的状态，否则，继续这个过程，直到leader被选举出来

	- fast paxos(默认)

		- fast paxos流程是在选举过程中，某Server首先向所有Server提议自己要成为leader，当其它Server收到提议以后，解决epoch和zxid的冲突，并接受对方的提议，然后向对方发送接受提议完成的消息，重复这个流程，最后一定能选举出Leader

- 节点类型

	- PERSISTENT-持久节点

		- 除非手动删除，否则节点一直存在于 Zookeeper 上

	- EPHEMERAL-临时节点

		- 临时节点的生命周期与客户端会话绑定，一旦客户端会话失效（客户端与zookeeper 连接断开不一定会话失效），那么这个客户端创建的所有临时节点都会被移除

	- PERSISTENT_SEQUENTIAL-持久顺序节点

		- 基本特性同持久节点，只是增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字

	- EPHEMERAL_SEQUENTIAL-临时顺序节点

		- 基本特性同临时节点，增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字

- watch机制

	- 原理

		- Zookeeper 允许客户端向服务端的某个 Znode 注册一个 Watcher 监听，当服务端的一些指定事件触发了这个 Watcher，服务端会向指定客户端发送一个事件通知来实现分布式的通知功能，然后客户端根据 Watcher 通知状态和事件类型做出业务上的改变

	- 特性

		- 一次性
		- 客户端串行执行
		- 轻量化

	- 机制

		- 客户端注册watcher

			- getData
			- exists
			- getChildren

		- 服务端处理watcher

			- create
			- delete
			- setData

		- 客户端回调watcher

## dubbo

### 原理

- RPC分布式服务框架

### 优点

- 核心业务不用多次开发
- 本地调用
- 分布式高性能高可用

### 核心功能

- Remoting

	- 网络通信框架，提供对多种NIO框架抽象封装，包括“同步转异步”和“请求-响应”模式的信息交换方式

- Cluster

	- 提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持

- Registry

	- 基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器

### 主要架构

- Provider
- Consumer
- Registry

	- 服务注册与发现的注册中心

- Monitor

	- 统计服务的调用次数和调用时间的监控中心

		- 功能

			- 服务查询，详情展示
			- 服务治理

				- 路由规则，条件路由，标签路由

			- 黑名单
			- 负载均衡

				- 随机
				- 轮训
				- 最小活跃

- Container

	- 服务运行容器

### 协议

- dubbo

	- 单一长连接和NIO异步通讯，适合大并发小数据量的服务调用，以及消费者远大于提供者。传输协议TCP，异步，Hessian序列化

		- dubbo协议做在会话层，下面使用的还是TCP\IP协议（传输层、网络层）
		- hessian反序列化如果找不到对应实现类会变成Map，所以调用方需要知道对应实现类

- http

	- 基于Http表单提交的远程调用协议，使用Spring的HttpInvoke实现。多个短连接，传输协议HTTP，传入参数大小混合，提供者个数多于消费者，需要给应用程序和浏览器JS调用

- redis

	- 基于redis实现的RPC协议

### 服务注册

- 扫描

	- xml配置

		- 通过META-INFO/spring.handlers配置的DubboNamespaceHandler扫描各种标签生成BeanDefinition

	- 注解扫描

- 生产者注册

	- 每个service方法的配置会解析成对应的com.alibaba.dubbo.config.spring.ServiceBean
	- 服务注册和暴露在方法afterPropertiesSet的export方法执行
	- 服务暴露

		- 最后跑到ServiceConfig.doExportUrlsFor1Protocol，使用protocol.export
		- 然后dubbo有对URL的protocol自适应扩展，在调用时根据url决定使用哪个扩展
		- 本地暴露时，protocol会转化为dubboProtocol，DubboProtocol.export方法通过netty开启服务，通过IP+port判断server是否存在，不存再就创建一个NettyServer来开启服务

	- 最后调用RegistryProtocol.doRegister注册服务

- 消费者注册

	- 解析<dubbo:reference/>时会实例会ReferenceBean对象，实例对象后同样会调用afterPropertiesSet方法。最终会调用RegistryProtocol的dorefer()方法进行服务注册
	- 服务发现

		- 注册之后注册目录(RegistryDirectory)订阅subscribeUrl的通知，此过程中会把invoker封装为invokerDelegate并在RegistryDirectory中缓存urlInvokerMap。这就完成了服务的发现

	- 最终会调用ZookeeperRegistry.doRegister注册服务

### 服务调用

- 在服务接口消费者初始化时，接口方法和提供者Invoker对应关系保存在RegistryDirectory的methodInvokerMap中，通过调用的方法名称（或方法名称+第一个参数）获得对应的提供者invoker列表，如注册中心设置了路由规则，对这些invoker根据路由规则进行过滤
- 读取到所有符合条件的服务提供者invoker之后，由LoadBalance组件执行负载均衡，从中挑选一个invoker进行调用，框架内置支持的负载均衡算法包括random（随机）、roundrobin（R-R循环）、leastactive（最不活跃）、consistenthash（一致性hash），应用可配置，默认random
- 在调用集群容错下的doInvoke，默认是FailoverClusterInvoker.doInvoke
- 最后NettyClient.send调用NettyChannel.send发送请求到生产者
- 服务端再响应

### 路由规则

- 解释

	- 路由规则在发起一次RPC调用前起到过滤目标服务器地址的作用，过滤后的地址列表，将作为消费端最终发起RPC调用的备选地址

- 类型

	- 条件路由(Condition Router)

		- 消费者配置字段

			- scope

				- 服务粒度(service)
				- 应用粒度(application)

			- key

				- 服务粒度(service)

					- [{group}:]{service}[:{version}]

				- 应用粒度(application)

					- application名称

			- conditions

	- 标签路由(Tag Router)

		- provider

			- 动态打标

				- 在服务治理控制台

					- 加入tags

						- 设置name和address

			- 静态打标

				- <dubbo:provider tag="tag1"/>
				- <dubbo:service tag="tag1"/>

		- consumer

			- RpcContext.getContext().setAttachment(Constants.REQUEST_TAG_KEY,"tag1")

	- 脚本路由(Script Router)

### 负载均衡

- Random LoadBalance(默认)

	- 随机选取提供者策略，有利于动态调整提供者权重。截面碰撞率高，调用次数越多，分布越均匀

- RoundRobin LoadBalance

	- 轮循选取提供者策略，平均分布，但是存在请求累积的问题

- LeastActive LoadBalance

	- 最少活跃调用策略，解决慢提供者接收更少的请求

- ConstantHash LoadBalance

	- 一致性Hash策略，使相同参数请求总是发到同一提供者，一台机器宕机，可以基于虚拟节点，分摊至其他提供者，避免引起提供者的剧烈变动

### 集群容错

- Failover Cluster(默认)

	-  失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟

- Failfast Cluster

	-  快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录

- Failsafe Cluster

	-  安全失败，出现异常时，直接忽略。通常用于写入审计日志等操作

- Failback Cluster

	-  失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作

- Forking Cluster

	-  并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks=”2″ 来设置最大并行数

### SPI

- 使用过程

	- 使用键值对将key和实现全限定名放在META-INFO/dubbo文件夹下，以接口类全限定名作为文件名称

		- optimusPrime=com.xx.xx.OptimusPrime

	- 在接口上使用@SPI
	- 使用ExtensionLoader

		- ExtensionLoader<Robot> extensionLoader = 
        ​    ExtensionLoader.getExtensionLoader(Robot.class);
		- Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();

- 扩展实现

### IOC

- 检查实现类是否有public修饰的set方法
- 通过ObjectFactory获取依赖对象，然后通过反射将依赖注入
- 通过自适应扩展点实现

### AOP

- 在META-INFO/dubbo文件夹下的对应配置文件中加入实现了接口的wrapper类全限定名

	- com.xx.xx.RobotWrapper

### 消费者线程池模型

- 2.7.5前

	- 业务线程发出请求，拿到一个 Future 实例。
	- 业务线程紧接着调用 future.get 阻塞等待业务结果返回。
	- 当业务数据返回后，交由独立的 Consumer 端线程池进行反序列化等处理，并调用 future.set 将反序列化后的业务结果置回。
	- 业务线程拿到结果直接返回

- 2.7.5后

	- 业务线程发出请求，拿到一个 Future 实例。
	- 在调用 future.get() 之前，先调用 ThreadlessExecutor.wait()，wait 会使业务线程在一个阻塞队列上等待，直到队列中被加入元素。
	- 当业务数据返回后，生成一个 Runnable Task 并放入 ThreadlessExecutor 队列
	- 业务线程将 Task 取出并在本线程中执行：反序列化业务数据并 set 到 Future。
	- 业务线程拿到结果直接返回

### 消费者提供者通信

- 服务暴露

	- 解析service标签，然后启用netty服务实现url的暴露

		- 在 Spring 实例化完 bean 之后，在刷新容器最后一步发布 ContextRefreshEvent 事件的时候，通知实现了 ApplicationListener 的 ServiceBean 类进行回调 onApplicationEvent 事件方法，Dubbo 会在这个方法中调用 ServiceBean 父类 ServiceConfig 的 export 方法，而该方法真正实现了服务的（异步或者非异步）发布

- 消费者通过netty client向netty server建立连接，boss负责建立，worker负责建立后的IO工作

	- 提供者的不同dispatcher设置

		- all（默认）

			- 全部消息发到线程池来处理

		- direct

			- 全部在IO线程上处理

- 消费者接到响应参考上面消费者线程池模型

### 注册中心宕机

- 注册中心对等集群，任意一台宕掉后，会自动切换到另一台
- 注册中心全部宕掉，服务提供者和消费者仍可以通过本地缓存通讯
- 服务提供者无状态，任一台 宕机后，不影响使用
- 服务提供者全部宕机，服务消费者会无法使用，并无限次重连等待服务者恢复

## solr

### 原理

- 基于lucene的全文检索服务器

### Inverted index(倒排索引，反向索引)

- 对全文进行分词然后作为索引

### 查询参数

- q – 查询字符串，必须的。Solr 中使用的基本查询。
- fq – （filter query）过虑查询，作用：在q查询符合结果中同时是fq查询符合的，
- sort – 排序，格式：sort=<field name>+<desc|asc>[,<field name>+<desc|asc>]… 。示例：（inStock desc, price asc）表示先 “inStock” 降序, 再 “price” 升序，默认是相关性降序。
- start – 返回第一条记录在完整找到结果中的偏移位置，0开始，一般分页用。
- rows – 指定返回结果最多有多少条记录，配合start来实现分页。
- fl- field作为逗号分隔的列表指定文档结果中应返回的 Field 集。默认为 “*”，指所有的字段。“score” 指还应 返回记分。

### 配置

- 单机

	- schema

		- name：指定域的名称
		- type：指定域的类型

			- StringField  不分词 索引 存储自定 常用于id，一些唯一的列
			- LongField    分词   索引 存储自定  常用于数字列
			- StoredField  不分词  不索引 存储      常用于图片，音频等 只需要展示的列
			- TextField    分词    索引   存储自定  适用于任何类型（solr 配置分词器的属性类型，一定是此 类型）

		- indexed：是否索引
		- stored：是否存储
		- required：是否必须
		- multiValued：是否多值

- 集群配合zk

	- collection：一个schema对应的索引集合

		- shard：collection的逻辑分片

			- leader
			- replication

## Lucene

### 索引里面究竟存些什么

- 倒排索引

### 如何创建索引

- 首先需要一些要索引的原文档
- 分词

	- 将文档分成一个一个单独的单词
	- 去除标点符号
	- 去除停词

- 将得到的词元传给语言处理组件

	- 变为小写
	- 将单词缩减为词根形式，如“cars”到“car”等
	- 将单词转变为词根形式，如“drove”到“drive”等

- 将得到的词(Term)传给索引组件

	- 利用得到的词(Term)创建一个字典
	- 对字典按字母顺序进行排序
	- 合并相同的词(Term)成为文档倒排(Posting List)链表

### 如何对索引进行搜索

- 用户输入查询语句
- 对查询语句进行词法分析，语法分析，及语言处理

	- 词法分析主要用来识别单词和关键字
	- 语法分析主要是根据查询语句的语法规则来形成一棵语法树
	- 语言处理同索引过程中的语言处理几乎相同，如learned变成learn等

- 搜索索引，得到符合语法树的文档
- 根据得到的文档和查询语句的相关性，对结果进行排序

	- 相关性计算权重(Term weight)的过程

		- Term Frequency (tf)：即此Term在此文档中出现了多少次。tf 越大说明越重要
		- Document Frequency (df)：即有多少文档包含次Term。df 越大说明越不重要

## ElasticSearch

### 索引层面调优手段

- 设计阶段调优

	- 根据业务增量需求，采取基于日期模板创建索引，通过 roll over API 滚动索引
	- 使用别名进行索引管理
	- 每天凌晨定时对索引做 force_merge 操作，以释放空间
	- 采取冷热分离机制，热数据存储到 SSD，提高检索效率；冷数据定期进行 shrink
	- 操作，以缩减存储
	- 采取 curator 进行索引的生命周期管理
	- Mapping 阶段充分结合各个字段的属性，是否需要检索、是否需要存储等

- 写入调优

	- 写入前副本数设置为 0
	- 写入前关闭 refresh_interval 设置为-1，禁用刷新机制
	- 写入过程中：采取 bulk 批量写入
	- 写入后恢复副本数和刷新间隔
	- 尽量使用自动生成的 id

- 查询调优

	- 禁用 wildcard
	- 禁用批量 terms（成百上千的场景）
	- 充分利用倒排索引机制，能 keyword 类型尽量 keyword
	- 数据量大时候，可以先基于时间敲定索引再检索
	- 设置合理的路由机制

### master 选举

- 流程

	- 确认候选主节点数达标，elasticsearch.yml 设置的值

		- discovery.zen.minimum_master_nodes

	- 比较：先判定是否具备 master 资格，具备候选主节点资格的优先返回；若两节点都为候选主节点，则 id 小的值会主节点。注意这里的 id 为 string 类型

- 特殊情况

	- 当集群master候选数量不小于3个时，可以通过设置最少投票通过数量(discovery.zen.minimum_ master_ nodes)超过所有候选节点一半以上来解决脑裂问题
	- 当候选数量为两个时，只能修改为唯一的一 -个master候选，其他作为data节点，避免脑裂问题

### 索引文档的过程

- 单文档

	- 第一步：客户写集群某节点写入数据，发送请求。（如果没有指定路由/协调节点，请求的节点扮演路由节点的角色。）
	- 第二步：节点 1 接受到请求后，使用文档_id 来确定文档属于分片 0。请求会被转到另外的节点，假定节点 3。因此分片 0 的主分片分配到节点 3 上。

		- 1shard = hash(_routing) % (num_of_primary_shards)

	- 第三步：节点 3 在主分片上执行写操作，如果成功，则将请求并行转发到节点 1和节点 2 的副本分片上，等待结果返回。所有的副本分片都报告成功，节点 3 将向协调节点（节点 1）报告成功，节点 1 向请求客户端报告写入成功

### 搜索的过程

- query then fetch

	- 假设一个索引数据有 5 主+1 副本 共 10 分片，一次请求会命中（主或者副本分片中）的一个
	- 每个分片在本地进行查询，结果返回到本地有序的优先队列中。
	- 第 2）步骤的结果发送到协调节点，协调节点产生一个全局的排序列表。
fetch 阶段的目的：取数据。
路由节点获取所有文档，返回给客户端

### 更新和删除文档的过程

- 删除和更新也都是写操作，但是Elasticsearch中的文档是不可变的，因此不能被删除或者改动以展示其变更;
- 磁盘上的每个段都有一个相应的.del文件。当删除请求发送后，文档并没有真的被删除，而是在.del文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del文件中被标记为删除的文档将不会被写入新段.
- 在新的文档被创建时, Elasticsearch会为该文档指定一个版本号, 当执行更新时，旧版本的文档在.del文件中被标记为删除，新版本的文档被索引到- .个新段.旧版本的文档依然能匹配查询，但是会在结果中被过滤掉

### 字典树

- 多叉树
- 每一个节点保存一个字符并且统计频率
- 空间换时间

### GC方面，在使用Elasticsearch时要注意什么

- 倒排词典的索引需要常驻内存，无法 GC，需要监控 data node 上 segmentmemory 增长趋势
- 各类缓存，field cache, filter cache, indexing cache, bulk queue 等等，要设置合理的大小，并且要应该根据最坏的情况来看 heap 是否够用，也就是各类缓存全部占满的时候，还有 heap 空间可以分配给其他任务吗？避免采用 clear cache 等“自欺欺人”的方式来释放内存。
- 避免返回大量结果集的搜索与聚合。确实需要大量拉取数据的场景，可以采用 scan & scroll api 来实现。
- cluster stats 驻留内存并无法水平扩展，超大规模集群可以考虑分拆成多个集群通过 tribe node 连接。
- 想知道 heap 够不够，必须结合实际应用场景，并对集群的 heap 使用情况做持续的监控。
- 根据监控数据理解内存需求，合理配置各类 circuit breaker，将内存溢出风险降低到最低

### 大数据量(上亿量级)的聚合如何实现

- Elasticsearch 提供的首个近似聚合是 cardinality 度量。它提供一个字段的基数，即该字段的 distinct 或者 unique 值的数目。它是基于 HLL 算法的。HLL 会先对我们的输入作哈希运算，然后根据哈希运算的结果中的 bits 做概率估算从而得到基数。其特点是：可配置的精度，用来控制内存的使用（更精确 ＝ 更多内存）；小的数据集精度是非常高的；我们可以通过配置参数，来设置去重需要的固定内存使用量。无论数千还是数十亿的唯一值，内存使用量只与你配置的精确度相关

### 在并发情况下，保证读写一致

- 可以通过版本号使用乐观并发控制，以确保新版本不会被旧版本覆盖，由应用层来处理具体的冲突；
- 另外对于写操作，一致性级别支持 quorum/one/all，默认为 quorum，即只有当大多数分片可用时才允许写操作。但即使大多数可用，也可能存在因为网络等原因导致写入副本失败，这样该副本被认为故障，分片将会在一个不同的节点上重建。
- 对于读操作，可以设置 replication 为 sync(默认)，这使得操作在主分片和副本分片都完成后才会返回；如果设置 replication 为 async 时，也可以通过设置搜索请求参数_preference 为 primary 来查询主分片，确保文档是最新版本

### 如何监控集群状态

- Marvel 让你可以很简单的通过 Kibana 监控 Elasticsearch。你可以实时查看你的集群健康状态和性能，也可以分析过去的集群、索引和节点指标

## 分布式锁

### redis

- setnx()+expire()

	- expire如果因为某些原因没有生效可能导致死锁

- setnx()+get()+getset()

	- setnx(key,当前时间加超时时间)
	- 如果setnx失败,get(key)获取超时时间，如果超时
	- 使用getset(key,新当前时间加超时时间)，并且对比是否是上一个时间，如果不是则再重复get()后的操作
	- 执行完然后再比较时间，如果是自己设置的时间则删除锁，不是自己设置的时间则不管

- redlock

	- 大于3的奇数个节点
	- 依次尝试从5个实例，使用相同的key和具有唯一性的value获取锁
当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间，这样可以避免客户端死等
	- 客户端使用当前时间减去开始获取锁时间就得到获取锁使用的时间。当且仅当从半数以上的Redis节点取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功
	- 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间，这个很重要
	- 如果因为某些原因，获取锁失败（没有在半数以上实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁，无论Redis实例是否加锁成功，因为可能服务端响应消息丢失了但是实际成功了，毕竟多释放一次也不会有问题

### zk

- 临时节点+watch机制

	- 原理

		- 每个锁占用一个普通节点 /lock，当需要获取锁时在 /lock 目录下创建一个临时节点，创建成功则表示获取锁成功，失败则 watch/lock 节点，有删除操作后再去争锁。临时节点好处在于当进程挂掉后能自动上锁的节点自动删除即取消锁

	- 缺点

		- 所有取锁失败的进程都监听父节点，很容易发生羊群效应

	- 优点

		- 非公平式竞争

- 临时有序节点+watch前一位机制

	- 优点

		- 避免了羊群效应

	- 缺点

		- 顺序获取锁，公平式的可能影响业务性能

	- 原理

		- 1、生成locknode/{guid}-lock-的znode，同时设置为EPHEMERAL_SEQUENTIAL
		- 2、尝试获取 locknode 下的所有节点，对其进行排序，若刚刚创建的节点处在第一位，则获取锁成功，退出当前流程
		- 3、若不为第一位，则对整个序列中排在自己持有的路径前一位的路径添加一个 watcher，并检查该前一位节点是否存在
		- 4、若前一位节点不存在，跳转至第二步，否则休眠等待。当被 watch 的路径发生变化时（通常是被删除），等待被唤醒并跳转至第二步

## Linux

### cpu问题

- top -H

### 磁盘

- df -mh

### 内存

- free

### 端口占用

- lsof -i [port]

### java堆内存和gc情况

- jstack -gcutil [pid]

### java进程CPU占用排查

- top命令查看进程ID
- 通过top -H -p pid查看进程下线程的cpu占用
- jstack pid > ./threadDump.log来记录堆栈信息
- 通过printf ‘%x\n’ pid得到 nid，在堆栈信息里查找具体操作

### java进程突然消失

- OOM导致的crash

	- 查看JVM参数 -XX:+HeapDumpOnOutOfMemoryError 和 -XX:HeapDumpPath=*/java.hprof
	- 根据HeapDumpPath指定的路径查看是否产生dump文件
	- 若存在dump文件，使用Jhat、MAT等工具分析即可

- jvm或者jdk本身的bug导致crash

	- 当JVM发生致命错误导致崩溃时，会生成一个hs_err_pid_xxx.log这样的文件
	- 默认情况下，该文件是生成在工作目录下的，当然也可以通过 JVM 参数指定生成路径：
-XX:ErrorFile=/var/log/hs_err_pid<pid>.log

- 系统的OOM-killer

	- 系统报错日志:/var/log/messages
	- egrep -i 'killed process' /var/log/messages

## Feed流解决方案

### 描述

- 本质上是一个数据流，是将 “N个发布者的信息单元” 通过 “关注关系” 传送给 “M个接收者”

### 数据层面

- 发布者数据
- 关注关系

	- 双向关注

		- 不会有大V，可以推模式解决

	- 单向关注

		- 会产生大V

- 接受者数据

### 方案

- 推模式
- 拉模式
- 推拉结合

### 可以考虑使用MyISam引擎

## UGC内容审核

### 审核机制

- 先发后审

	- 未命中屏蔽词先发，加入审核队列，由审核人员审核

- 先审后发

	- 数量少的内容可以这样做

### 实现的方式

- 根据人群进行分类审核

	- 新用户
	- 普通用户
	- 大V
	- 高危用户

- 机审规则

	- 建立并完善屏蔽词库
	- 建立违禁号码库或者对所有号码进行屏蔽
	- 重复内容过滤
	- 限制用户发布次数，避免灌水
	- 其他限制条件，如绑定手机号，用户注册一段时候后才能发布信息

- 人审

	- 主要针对视频或者图片
	- 一些中文谐音垃圾信息，并加入屏蔽词库
	- 增加举报机制，培养用户举报习惯

## 突发rt长问题

### 网络波动

### 懒加载

### 发生fullgc

### 缓存失效

### JIT未预热

## 用户认证

### Session

- 存放一个SessionId到cookie，如果cookie禁用，可以加在URL里面重定向
- 服务端通过SessionId查询到保存在内存中的具体数据
- 分布式系统下需要共享session数据，有同步问题
- 默认过期时间20分钟

### JWT

- 服务端直接将用户认证信息加密全部存给客户端，最好存放在localstorage
- 发送给服务端时存放在Header的Authorization中，使用Bearer模式添加
- 分布式下可以直接跨域访问应用

### Token

- 服务端签发一个token，客户端访问携带
- 服务端直接验证token的合法性

## OSI(open system interconnection model)开放系统交互模型

### 应用层

- HTTP、HTTPS、TELNET、SSH、SMTP、POP3
- 为应用软件设计的接口

### 表示层

- 把数据转为能与接受者系统兼容并适合传输的格式

### 会话层

- 在数据传输中设置和维护计算机网络中两台计算机之间的通信连接

### 传输层

- 把传输表头加至数据形成数据包。表头包含所使用协议，如TCP协议

### 网络层

- 决定数据的路径和转寄，将网络表头加至数据包，形成分组。如IP协议

### 数据链路层

- 负责网络寻址、错误侦测和改错（地址解析，反向解析，转发）。例如以太网、wifi

### 物理层

- 底层发送数据帧的硬件，如网卡

## TCP\IP协议分层

### 应用层

### 传输层

### 网络层

### 网络接口层

## 设计模式

### 原则

- 开闭原则

	- 对扩展开放，对修改关闭

- 里式替换原则

	- 子类可以替换基类使用

- 依赖倒转原则

	- 针对接口变成，依赖于抽象

- 接口隔离原则

	- 使用多个隔离的接口，比单个接口好，功能的解耦

- 最少知道法则

	- 实体尽量少的和其他发生相互作用，是功能模块相对独立

- 合成复用原则

	- 尽量使用合成\聚合的方式，而不是使用继承

### 常见设计模式

- 单例模式

	- 主要是volatile+synchronized的双检锁

- 工厂模式

	- 根据工厂创建不同的实例对象

- 抽象工厂模式

	- 有一个工厂抽象类，可以创建不同的工厂

- 策略模式

	- if的升级，提高代码可读性

- 责任链模式

	- 基于抽象类和指针

- 观察者模式

	- 被观察者持有观察者实例，然后可以在被观察者中视情况调用观察者

- 模版模式

	- 定义抽象类，子类自己去实现

- 代理模式

	- 代理类和被代理类实现同一个父类，代理类持有被代理类，然后调用代理类，同时可以做业务增强

## 事务

### 本地多数据源

- 先设置事务ID，通过threadlocal记录操作内的connection，然后执行操作，如果抛出异常，则改线程记录的所有connection roll back

### 分布式事务

- 实现

	- seata

		- XA
		- AT

			- 基于sql解析实现了补偿机制，比XA在高并发下性能更好

- 方法

	- 2PC-2阶段提交

		- prepare

			- TC写本地日志，并持久化。TC向所有参与者发送Prepare T消息
			- 各参与者TM收到Prepare T消息，根据自身情况，决定是否提交事务

				- 如果决定提交，TM写日志并持久化，向TC发送Ready T消息
				- 如果决定不提交，TM写日志并持久化，向TC发送Abort T消息，本地也进入事务abort流程

		- commit/rollback

			- 当TC收到所有节点的回应，或者等待超时，决定事务commit或abort

				- 如果所有参与者回应Ready T，则TC先写日志并持久化，再向所有参与者发送Commit T消息
				- 如果收到至少一个参与者Abort T回应，或者在超时时间内有参与者未回应，则TC先写日志，再向所有参与者发送Abort T消息

			- 参与者收到TC的消息后，写或日志并持久化

		- 异常处理

			- 协调者正常，参与者宕机

				- 发生在第一阶段：由于协调者无法收集到所有参与者的反馈，会陷入阻塞情况，超时都未回复的话发送终止事务
				- 发生在第二阶段：无论协调者发起的是提交还是终止，那宕机的参与者在重启之后，都将执行对应操作，不存在不一致情况

			- 协调者宕机，参与者正常

				- 参与者没有备份协调者

					- 等待协调者回复

				- 参与者备份了协调者

					- 检查所有参与者存在commit或者rollback，所有参与者均可以执行commit或者rollback
					- 所有参与者均没有commit或者rollback，可以继续也可以abort，建议abort释放资源

			- 协调者、参与者都发生了宕机

				- 参与者没有备份协调者

					- 等待协调者回复

				- 参与者备份了协调者

					- 全部参与者未prepare，可以继续也可以abort，建议abort释放资源
					- 检查所有参与者存在commit或者rollback，所有参与者均可以执行commit或者rollback
					- 全部参与者prepare，但没有commit或者rollback，等待宕机的参与者恢复

	- 3PC-3阶段提交

		- 与2PC相比

			- 2PC的prepare阶段是执行事务不提交，3PC的canCommit不执行事务，仅做准备，preCommit才是执行事务
			- preCommit阶段增加了参与者超时机制，超时后本地commit，避免一直阻塞，但是仍有数据不一致问题

	- TCC

		- 业务实现方要实现三个接口

			- Try

				- 将数据进行准隔离，例如预扣库存，预扣金额

			- Confirm

				- 进行执行

			- Cancel

				- 释放资源

		- 调用方也需要配合访问3个接口
		- 异常情况

			- 操作

				- 无论comfirm还是cancel都重试

			- 场景

				- 空回滚

					- 现象

						- try异常，以为调用了却没有锁定资源，超时后调用cancel

					- 解决方法

						- cancel时判断是否有try

				- 幂等

					- 现象

						- 提交阶段异常时可能会重复调用 confirm 和 cancel，所以要实现幂等，保证多次执行效果一致

					- 解决方法

						- 记录事务执行状态，如果执行过了，就不再执行

				- 悬挂

					- 现象

						- 现象是先执行了 cancel，后执行的 try，造成资源没人释放

					- 解决方法

						- 还是记录事务执行状态，try 执行时判断 cancel 是否执行了

	- 最终一致性

		- 例如rocketmq的事务消息

			- 用户注册成功后发送邮件、电商系统给用户发送优惠券

	- 本地消息表
	- 最大努力通知

		- 最简单的一种柔性事务，适用于一些最终一致性时间敏感度低的业务，动方处理结果 不影响主动方的处理结果

			- 支付回调

### 本地单数据源事务

- Spring的@transaction

## SpringCloud

### 服务发现

- Eureka(2.x已经闭源,1.x还在维护)

	- 客户端如何注册

		- 客户端启动时创建定时任务，定时向服务端发送心跳，如果服务端响应为404则发送注册请求

	- 服务端如何保存注册信息

		- 服务端将客户端注册信息保存到一个ConcurrentHashMap中

	- 客户端如何拉取服务端已保存的服务信息

		- 客户端拉取服务端服务信息是通过一个定时任务定时拉取的，每次拉取后刷新本地已保存的信息，需要使用时直接从本地直接获取

	- 高可用配置和同步

		- 配置ureka.client.service-url.defaultZone，后面放入地址即可
		- 收到客户端注册信息会同步到其他注册中心，如果服务端收到服务端的注册信息只同步，不再传播

	- 服务剔除机制

		- 如果15分钟内超过85%的客户端节点都没有正常的心跳，那么这个服务端开启自我保护模式，不剔除服务
		- 没有开启自我保护则用随机数的方式将这些已过期的实例进行剔除

- Netfilix
- Nacos

### rest客户端

- Feign
- OpenFeign

### API网关

- Gateway
- Zull

### 配置管理

- Config
- Nacos

### 权限

- security
- outh2
- JWT

### 限流

- sentinel

### 分布式事务

- seata

### 限流

- Hystrix

	- 状态

		- 打开

			- 一定次数无法调用且多次检测没有恢复迹象，打开后无法请求到该服务

		- 半开

			- 短时间有恢复迹象，部分请求发过来

		- 关闭

			- 正常访问

	- 服务熔断

		- 原理

			- 调用失败可以调用本地定义好的熔断方法

		- 情况

			- 设定一定时间内的调用次数超过了设定次数，调用熔断方法
			- 请求超时熔断

	- 服务隔离

		- 线程池+semaphore

- sentinel

## SpringBoot

### 启动

- 注解

	- @SpringBootApplication

		- @EnableAutoConfiguration

			- 启用 SpringBoot 的自动配置机制

		- @Configuration

			- 允许在上下文中注册额外的 bean 或导入其他配置类

		- @ComponentScan

			- 默认扫描同级包和子包的配置

- main方法run()

	- AnnotationConfigServletWebServerApplicationContext和AnnotationConfigApplicationContext区别在于ServletWebServer，Springboot自己在refresh()的onFresh()中启动webServer，其他还是一样的流程

### 打包jar包后修改配置文件

- 当前目录下的/config目录
- 当前目录
- classpath里的/config目录
- classpath 根目录

### 自动装配

- 关注@SpringBootApplication下的@EnableAutoConfiguration

	- 实现通过AutoConfigurationImportSelector

		- 扫描META-INF/spring.factories中是否配置了EnableAutoConfiguration

			- 引入spring-boot-starter
			- 写一个@Configuration配置类，自己实现一些条件下需要自动装配的类
			- 配置META-INF/spring.factories，设置EnableAutoConfiguration=配置类全限定名

## SpringMVC

### 请求响应流程

- 请求至前端控制器DispatcherServlet
- DispatcherServlet调用HandlerMapping处理器映射器，请求获取Handler
- HandlerMapping根据url找到具体处理器，生成处理器以及拦截器，返回给DispatcherServlet
- DispatcherServlet调用HandlerAdapter
- HandlerAdapter经过适配调用具体的Handler
- Handler执行完后返回ModelAndView
- HandlerAdapter将ModelAndView返回给DispatcherServlet
- DispatcherServlet将ModelAndView传给ViewResolver进行解析
- 解析后返回View
- DispatcherServlet对View进行数据填充
- 响应用户

### @RestController

- =@Controller+@ResponseBody

	- ResponceBod相当于直接把数据放到整个body中，而不需要渲染页面

### xml启动流程

- 通过配置文件的ContextLoaderListener启动
- 调用父类ContextListener的initWebApplicationContext方法

	- 检查容器是否已经存在
	- 创建容器

		- 判断用户是否有自定义容器
		- 没有则使用默认容器（xmlWebApplicationContext）

- 判断容器是否是ConfigurableWebApplicationContext
- 刷新并在servlet中设置自己为RootWebApplicationContext
- 返回创建好的容器

### java-config启动

- AnnotationConfigApplicationContext

## Spring

### 启动过程

- 继承于AbstractApplicationContext

	- AnnotationConfigApplicationContext

		- this()无参构造器

			- 父类中GenericApplicationContext会实例化一个DefaultListableBeanFactory工厂
			- 实例化一个BeanDefinitionReader读取器

				- 添加两个beanFactory后置处理器

					- ConfigurationClassPostProcessor

						- 完成 bean 的扫描与注入工作

					- EventListenerMethodProcessor

				- 添加两个Bean后置处理器

					- AutowiredAnnotationBeanPostProcessor

						- 用来完成 @AutoWired 自动注入

					- CommonAnnotationBeanPostProcessor

				- 添加普通组件

					- DefaultEventListenerFactory

			- 实例化一个ClassPathBeanDefinitionScanner扫包器

				- 顾名思义根据配置的扫描位置去决定要扫描那些包和文件

		- reigister(annotatedClasses)

			- doRegisterBean()将配置信息注册到容器中，只是注册BeanDefinition信息，未实例化

				- BeanDefinition：不仅包含class，还包含了在spring的Bean的生命周期相关参数，如是否懒加载，是否单例等

		- refresh()

			- prepareRefresh()

				- 刷新前的处理

			- obtainFreshBeanFactory()

				- 获取beanFactory

			- prepareBeanFactory(beanFactory)

				- beanFactory预处理

			- postProcessBeanFactory(beanFactory)

				- 子类可以重写这个方法在beanFactory创建和准备好后做进一步处理

			- invokeBeanFactoryPostProcessors(beanFactory)

				- 执行beanFactory后置处理器

			- registerBeanFactoryPostProcessors(beanFactory)

				- 向容器注册beanFactory后置处理器

			- initMessageSource()

				- 初始化MessageSource组件：国际化、消息绑定和解析

			- initApplicationEventMulticaster()

				- 初始化事件派发器，在注册监听器时会用到

			- onRefresh()

				- 子类自己去实现一些操作，web场景会用到

			- registerListeners()

				- 注册监听器

			- finishBeanFactoryInitialization(beanFactory)

				- 初始化所有实例bean

			- finishRefresh()

				- 通知完成了刷新

	- ClassPathXmlApplicationContext

		- 同样的会调用refresh()

			- 和注解不同的是

				- 获取beanFactory时会使用AbstractRefreshableApplicationContext中的refreshBeanFactory()创建一个beanFactory并且根据xml设置beanDefinition

### bean的生命周期

- 实例化BeanFactoryPostProcessor并调用postProcessBeanFactory方法

	- 扫描、注册BeanDefinition

- 实例化BeanPostProcessor
- 实例化InstantiationAwareBeanPostProcessorAdapter
- 执行InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation
- 执行Bean构造器实例化
- 执行InstantiationAwareBeanPostProcessor的postProcessPropertyValues

	- 例如AutowiredAnnotationBeanPostProcessor实现autowired注入

- Bean属性注入
- 实现了BeanNameAware的话，会调用setBeanName()

	- 设置Bean的ID

- 实现了BeanFactoryAware的话，会调用setBeanFactory()

	- 传入beanFactory

- 实现了BeanPostProcessor的话，会有一个postProcessBeforeInitialization()可以使用

	- 例如这里标记了出不需要被代理的Bean，比如AOP的基础设施类

- 实现了InitializingBean的话，会调用afterPropertiesSet()
- 调用Bean的initMethod方法
- 实现了BeanPostProcessor的话，会调用postProcessAfterInitialization()

	- 例如AOP在这里通过AnnotationAwareAspectJAutoProxyCreator实现

- 调用InstantiationAwareBeanPostProcessor的postProcessAfterInitialization
- 容器关闭时调用DisposableBean的destroy()
- 调用Bean的destroyMethod方法

### @Autowired

- 原理

	- 启动容器的时候会注册一个AutowiredAnnotationBeanPostProcessor

		- 创建bean时（实例化对象和初始化），会调用各种BeanPostProcessor对bean初始化，AutowiredAnnotationBeanPostProcessor负责将相关的属性依赖注入进来

- 和@Resource的区别

	- 相同点

		- 都可以byType

	- 不同点

		- 来源

			- @Autowired来自Spring
			- @Resource来自JDK1.6以上

		- 注入方式

			- @Autowired只按照byType

				- 多个实现子类的时候需要配合Qulifier

			- @Resource默认按照byName，没有匹配到再byType

### 循环依赖

- 3级缓存

	- Map<String,ObjectFactory<?>>singletonFactories三级缓存，对象工厂
	- Map<String,Object>earlySingletonObjects二级缓存，实例化，属性填充，未初始化
	- Map<String,Object>singletonObjects一级缓存，实例化以及初始化

- 过程

	- A现在对象工厂中
	- 发现依赖了B，去实例化B
	- 创建B的过程中发现依赖A，那就从对象工厂获取A，然后把A放入二级缓存，从三级缓冲中删除，这时B就初始化好了，放入一级缓存
	- A再回来获得B，完成初始化，把A放入一级缓存，从二级缓存中删除

- 扩展问题

	- 为什么要使用三级缓存呢，二级缓存能解决循环依赖吗？

		- 如果要使用二级缓存解决循环依赖，意味着所有Bean在实例化后就要完成AOP代理，这样违背了Spring设计的原则，Spring在设计之初就是通过AnnotationAwareAspectJAutoProxyCreator这个后置处理器来在Bean生命周期的最后一步来完成AOP代理，而不是在实例化后就立马进行AOP代理

### Inversion of Control

- 一种思想
- 借助于“第三方”实现具有依赖关系的对象之间的解耦

### Dependency Injection

- 一种设计模式，一种IoC的实现
- 通过Spring IOC容器来创建对象，实现对象的注入，以及对对象生命周期的管理

### 设计模式

- BeanFactory的工厂模式

	- 生产和管理多个Bean

		- 和FactoryBean的区别

			- FactoryBean用工厂模式和修饰器模式增强生产一个类

- Bean默认的单例模式

	- 线程安全的方式

		- volatile+synchronized的双检锁

			- 使用volatile的原因：创建对象有3步

				- 分配内存空间
				- 初始化对象
				- 把内存空间的地址赋值给对象的引用
				- 如果不加volatile，那么jvm在创建时可能产生指令重排，导致对象已经有内存空间但是没有初始化好

		- 静态内部类，构造函数要私有
		- 枚举类，默认线程安全，还可以防止反序列化重新创建新对象

- AOP用的JDK动态代理和cglib字节码生成技术的代理模式
- Spring的Listener的实现ApplicationListener的观察者模式

### AOP

- 动态代理

	- 对于实现了接口的，Spring AOP会使用JDK Proxy，去创建代理对象，后面直接调用代理对象

		- 通过reflect包下的Proxy使用newProxyInstance传入被代理类的class的interfaces和实现了InvocationHandler类的class和类生成代理类

	- 对于没有实现接口而是继承了类的对象，Spring AOP会使用Cglib，生成一个被代理对象的子类，后面直接调用子类

		- 通过cglib包下的Enhancer设置被代理类为父类，然后设置实现了MethodInterceptor的类为回调，生成代理类后再强制类型转换为被代理类

- 概念

	- 通知Advice

		- 前置@Before
		- 后置@After
		- 环绕@Around
		- 返回@AfterReturning
		- 异常@AfterThrowing

	- 连接点Join Point

		- 在Spring AOP中就是方法的调用

	- 切点Pointcut

		- 连接点的集合

	- 切面Aspect
	- 目标对象Target Object
	- 代理对象AOP porxy

- 失效场景

	- 没有使用@transactional的方法调用了类内部的@transactional方法
	- 解决方法

		- 不要放到同一个类
		- 使用((xxxService)AOPContext.currentProxy).method()来调用代理类

### 事务

- 传播机制

	- PROPAGATION_REQUIRED

		- 默认，当前有就用，没有就新建

	- PROPAGATION_SUPPORT

		- 没有就不管，有就用

	- PROPAGATION_MANDATORY

		- 必须在一个事务中运行，如果没有事务，将抛出异常

	- PROPAGATION_REQUIRES_NEW

		- 必须运行在自己的事务里，互相独立

	- PROPAGATION_NOT_SUPPORTED

		- 不运行在事务里，有事务也不管

	- PROPAGATION_NEVER

		- 不能存在事务，如果存在事务，将抛出异常

	- PROPAGATION_NESTED

		- 如果有事务，运行在嵌套事务里，内部不影响外部，但是外部回滚内部要回滚；没有事务时和required一样

- 原理

	- 通过threadlocal管理数据源以及连接，根据策略选择是和否使用该连接

- 问题

	- 多数据源下使用transaction会导致数据源无法切换

- @transactional使用了cglib

	- 因为transactionManagerInterceptor实现的是MethodInterceptor
	- 没使用jdk动态代理，service不一定是个接口

- 失效的场景

	- 数据库不支持事务，如mySIAM
	- 事务类没有被Spring管理
	- 事务内部catch了异常
	- 注解在了非public方法，spring做的判断
	- 调用内部方法，内部方法不会使用代理类
	- rollbackFor设置错误
	- 事务传播属性设置错误

## Java

### 三大特性

- 封装

	- 主要是隐藏内部的属性或者方法，通过访问修饰符控制，外部只管用，不要知道细节

		- public

			- 所有类

		- protected

			- 不同包的子类以及向下包容

		- default

			- 同一个包的类以及向下包容

		- private

			- 同一个类

- 继承

	- 抽象类
	- 接口

- 多态

	- 继承
	- 重写overriding
	- 重载overloading

		- 方法类型和顺序参数不同

	- 向上转型

### 范型与类型擦除

- 范型

	- 解释

		- 参数化类型

	- 特点

		- 编译时校验
		- 编译时检查后擦除后变为原始类型

	- 应用

		- 范型类

			- List<T>
			- Map<T>
			- MyInterator<? super Integer>

		- 范型接口

			- Iterator<T>

		- 范型方法

			- public <T> T myMethod()

	- 好处

		- 与Object相比，范型使数据类型可以由外部传入，提供了扩展能力
		- 提供了类型检测机制
		- 提升了可读性

- 类型擦除

	- 解释

		- 使用泛型的时候加上类型参数，在编译器编译的时候会去掉，这个过程

	- 特点

		- 擦除后只保留原始类型，可以通过反射在ArrayList<Integer>里add("abc");

			- 原始类型

				- object或者extends确定的上边界

	- 问题

		- 先检查再编译以及编译的对象和引用传递问题

### JVM

- 组成

	- 运行时数据区

		- 本地方法栈(Native Method Stack)

			- 线程私有
			- 与虚拟机用到的 Native 方法相关

		- 程序计数器(Program Counter Register)

			- 线程私有
			- 倘若当前执行的是 JVM 的方法，则该寄存器中保存当前执行指令的地址；倘若执行的是native 方法，则PC寄存器中为空。
			- 作用，原因

				- CPU会在不同线程中切换，所以需要指令地址和栈帧来保证恢复现场

		- 虚拟机栈(VM Stack)

			- 线程私有
			- 每个方法在执行时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。栈的大小可以固定也可以动态扩展
			- 异常StackOverflowError

		- 堆(Heap)

			- 堆结构

				- Young Generation

					- Eden Space
					- Survivor Space

						- S0
						- S1

				- Old Space

			- 特点

				- 线程共享
				- 在虚拟机启动的时候就已经创建。所有的对象和数组都在堆上进行分配。字符常量池也在1.8后移到了这里，原来在永久代

		- 方法区(Method Area)

			- 线程共享
			- 1.7时是永久代，方法区逻辑上属于堆的一部分。主要用于存储类的信息、常量池、方法数据、方法代码等，但是大小有限，容易OOM。
			- 1.8时移除，放入metaSpace，使用本地内存。主要存放类信息，运行时常量池，JIT优化编译后的代码。本地内存够大就不会OOM，也可以通过参数设置。

	- 常量池

		- class常量池
		- 运行时常量池
		- 全局字符串常量池
		- 基础类型常量池

	- 执行引擎

		- 即时编译器JITCompiler

			- 基础概念

				- 解释执行

					- 将字节码(class)翻译成机器语言执行

				- 编译执行

					- 将字节码优化编译成本地机器代码

			- 优点

				- 对常用代码，如经常访问的方法和循环进行编译
				- 逃逸分析下对象可能被标量替换放在栈帧
				- 方法内联减少开支

			- 热点代码采样方法

				- 基于采样

					- 原理

						- 周期性检查栈顶，对经常出现的方法进行优化

					- 缺点

						- 有些线程会因为阻塞在栈顶导致误判

				- 基于计数器Hotspot使用

					- 原理

						- 统计方法或者代码块执行次数，超过一定次数触发JIT

					- 结构

						- 方法调用计数器（Invocation counter）

							- 记录方法调用次数，达到限度触发JIT编译，一段时间没有达到则发生衰减，计数减少一半

						- 回边计数器（Back Edge counter）

							- 记录方法体中循环代码执行的次数，达到限度触发OSR编译，不会衰减

		- 垃圾回收器GC

			- GC Roots

				- 能作为gc roots的对象

					- 本地方法栈中的引用对象（Native 方法的引用对象）
					- 虚拟机栈帧上本地变量表中的引用对象（方法参数、局部变量、临时变量）
					- 方法区中的静态属性引用类型对象、常量引用对象
					- Java 虚拟机内部的引用对象，如异常对象、系统类加载器等
					- 被同步锁（synchronize）持有的对象
					- Java 虚拟机内部情况的注册回调、本地缓存等

				- 三色标记算法

					- 基本概念

						- GCRoot根据可达分析算法遍历引用链 ,按照是否访问过该对象分成三种不同的颜色：白色、灰色、黑色。
						- 白色：本对象没有被访问过 （没有被GCRoot扫描过，有可能是为垃圾对象）
						- 灰色：本对象已经被访问过（被GCRoot扫描过），且本对象中的属性没有被GCRoot扫描，该对象就是为灰色对象；如果该对象的属性被扫描了，从灰色变为黑色
						- 黑色：本对象已经被访问过（被GCRoot扫描过），且本对象中的属性已经被GCRoot扫描过，该对象就是为黑色对象

					- 原理

						- 初始状态，所有节点为白色
						- 初始标记，gc roots直接引用对象标记为灰色
						- 并发标记，扫描整个引用链，有子节点的话，本节点标记为黑色，子节点标记为灰色，重复该步骤直至没有灰色标记
						- 白色节点即可能被垃圾回收器回收

					- 漏标、误标问题

						- 问题描述

							- 并发标记时可能因为用户线程导致扫描出现标记了不该标记的（误标），没标记该标记的（漏标）

								- 误标没有大影响，在下一次gc前只会以浮动垃圾存在
								- 漏标就影响程序的正确性了，会导致不该被回收的可能被回收掉

									- 漏标条件

										- 1、灰色对象不再引用白色对象
										- 2、黑色对象引用白色对象

						- 解决方法

							- CMS

								- 写栅栏 + 增量更新

									- 写栅栏

										- 类似AOP，在写操作前后加入pre和post操作

									- 增量更新

										- 写入时记录新增引用，重新标记时再扫描

											- 从条件二破坏漏标

							- G1

								- 写栅栏 + SATB

									- 写栅栏

										- 类似AOP，在写操作前后加入pre和post操作

									- SATB

										- Snapshot At The Beginning:记录最初gc roots确定时的快照

											- 从条件一破坏漏标

							- ZGC

								- 读栅栏

									- 类似AOP，在读操作前后加入pre和post操作

										- 从条件二破坏漏标

			- 回收算法

				- 标记清除

					- 通过根节点（GC Roots）标记所有从根节点开始的对象，没有被标记的就是没有被引用的，在清除阶段回收

				- 标记整理

					- 分为标记和整理两个阶段：首先标记出所有需要存活的对象，让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存

				- 复制算法

					- 从根集合节点进行扫描，标记出所有的存活对象，并将这些存活的对象复制到一块儿新的内存上去，之后将原来的那一块儿内存全部回收掉

			- 触发情况

				- 手动调用，不一定立刻gc
				- 老年代满了
				- 方法区满了

			- 垃圾收集器

				- 年轻代

					- Par New

						- 多线程的Serial

					- Parallel Scavenge

						- 可控制的吞吐量

				- 老年代

					- Parallel Old

						- 标记整理

					- CMS（Concurrent Mark Sweep）

						- 标记清除
						- 优点

							- 并发收集，低停顿

						- 缺点

							- CPU敏感
							- 无法处理浮动垃圾

						- 过程

							- 初始标记STW
							- 并发标记
							- 重标记STW
							- 并发清理

				- 分区回收

					- G1

						- 原理

							- 默认将堆空间规划为2048个区域，收集时不用全堆进行，同时计算每个区域的收集成本并量化，因此停顿时间可以预测
							- 标记整理

						- 过程
						- 模式

							- young gc

								- 选定所有年轻代

							- mixed gc

								- 选定所有年轻代和部分老年代

							- serial full gc

						- 卡表

							- 场景

								- 老年代引用了年轻代的对象，那么这个年轻代可能作为GC Root然后去扫描全堆。

							- 解决方案

								- 将堆划分为512字节的卡，然后每个卡有一个标志位来表示是否可能指向年轻代的引用，如果存在标识位脏卡，在minor gc的时候就可以把脏卡的对象加入分析，避免全堆扫描，在复原标识位

				- 使用组合

					- -XX:UseParallelGC(1.8默认)

						- Parallel Scavenge+ParallelOld

					- -XX:UseConcMarkSweep

						- Parallel New+CMS+Serial Old

					- -XX:UseG1GC

						- G1+Serial Old

	- 类加载器

		- 类加载过程

			- 加载

				- 双亲委派机制

					- 描述

						- 某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载

					- 特点

						- 如果同样加载一个System类，A已经加载的情况下，B就不会再加载了

					- 作用

						- 防止class重复加载
						- 保证核心类的安全

					- 如何打破

						- 自定义类加载器重写loadClass方法
						- SPI则用Thread.currentThread().getContextClassLoader()来加载实现类，实现在核心包里的基础类调用用户代码

			- 链接

				- 验证

					- 保证加载进来的字节流符合虚拟机规范

				- 准备

					- 类变量（注意，不是实例变量）分配内存，并且赋予初值

				- 解析

					- 常量池内的符号引用替换为直接引用的过程

			- 初始化

				- 类的初始化

		- 类型

			- 根加载器BootstrapClassLoader

				- java类加载的顶层加载器，负责核心类的加载，是扩展类加载器的父加载器

			- 扩展类加载器ExtClassLoader

				- java扩展类库的类加载器，是应用类加载器的父加载器

			- 应用类加载器AppClassLoader

				- 一般用户使用的类加载器。一般也是自定义类加载器的父加载器

			- 自定义类加载器

	- 本地方法接口
	- 本地方法库

- 内存流动

	- Eden Space

		- S0、S1中跳转

			- Old Space

### JMM

- happens-before规则

	- 程序顺序规则

		- 一个线程中，按照程序顺序，前面的操作 Happens-Before 于后续的任意操作

	- 监视器锁规则

		- 对一个锁的解锁，happens-before于随后对这个锁的加锁

	- volatile变量规则

		- 对一个volatile域的写，happens-before于任意后续对这个volatile域的读

	- 传递性

		- 如果A happens-before B，且B happens-before C，那么A happens-before C

	- start()规则

		- 它是指主线程 A 启动子线程 B 后，子线程 B 能够看到主线程在启动子线程 B 前的操作

	- join()规则

		- 如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回

### 反射

- 获得反射的方式

	- 实例的getClass方法
	- Class.forName(“全限定名”)
	- 类名.class

- 反射后实例化

	- //获取有参构造
		Constructor c = clazz.getConstructor(String.class,int.class);	
//通过有参构造创建对象
Person p = (Person) c.newInstance("张三",23);
		//获取eat方法							
Method m=clazz.getMethod("eat");
//执行方法
m.invoke(p);
		//获取有参eat方法		
Method m2=clazz.getMethod("eat",int.class);
//执行方法
m2.invoke(p,3);


- 效率低的问题

	- Method#invoke 方法会对参数做封装和解封操作
	- 需要检查方法可见性
	- 需要校验参数
	- 反射方法难以内联
	- JIT 无法优化

### 集合

- HashMap

	- 1.8前使用头插法，在高并发扩容时链表可能产生循环，使用get方式形成死循环；1.8后使用尾插法避免该问题

		- 1.7

			- next=e.next;
e.next=newTable[i];
newTable[i]=e;
e=next;

		- 1.8

			- 利用按位与的方式获得该元素在新map中的位置

	- 负载因子0.75的原因：过大会导致空间不能完全利用，效率低下；过小导致空间频繁扩容；根据泊松分布计算得来，在时间和空间上比较适合的值。
	- 容量是如何计算到2的次幂的

		- 把传入的cap-1=n
		- 然后n=n|n>>>1
		- 然后n=n|n>>>2
		- 然后n=n|n>>>4
		- 然后n=n|n>>>8
		- 然后n=n|n>>>16
		- 最后再返回n+1

	- 构成

		- 数组

			- 链表

				- 红黑树（最高和最低的层高不超过2，通过旋转平衡实现；二叉树在极端情况下就是链表）

					- 每个节点是红色或者黑色
					- 根节点为黑色
					- 每个为空叶子节点是黑色
					- 红色节点的子节点给黑色
					- 一个节点到子孙节点包含相同黑色节点

	- 扩容原理

		- 用key的hashCode的高位与index-1进行与运算往数组里放，如果已经存在，通过equals比较，相同覆盖，不同以链表方式加在后面，链表长度达到8后转化为红黑树

- ConcurrentHashMap

	- 1.7

		- 分段锁segment，最多16个锁

			- segment是ReentrantLock的子类

	- 1.8

		- CAS数组上的节点
		- Synchronized链表或者红黑树的首节点
		- 并发扩容原理

			- 根据CPU数，数组长度计算出每个线程每次可以处理的数组数，最小16个
			- 如果新table为空，对其初始化
			- 使用advanced，finishing来标记是否可以移至下一个数组，和是否结束扩容
			- 数据迁移时锁住头节点，按照hash和原length做位运算，为1放在高位，为0放在低位

- 快速失败

	- java.util下的都是

		- 迭代集合的时候对集合进行了增删改，那么检测到modCount与预期的不同就会抛出concurrentModificationException

- 安全失败

	- java.util.concurrent下的都是

		- 遍历的是集合的拷贝，就避免了抛出快速失败下的异常，但是并发修改不能及时检测到

### 线程

- 创建方式

	- 继承Thread
	- 实现Runable
	- 实现Callable

		- Future来接
		- FutureTask来接

- 状态

	- 初始(NEW)：新创建了一个线程对象，但还没有调用start()方法
	- 运行(RUNNABLE)：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”
	- 阻塞(BLOCKED)：表示线程阻塞于锁
	- 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）

		- wait()
		- join()

	- 超时等待(TIMED_WAITING)：该状态不同于WAITING，它可以在指定的时间后自行返回

		- sleep(long)
		- wait(long)

	- 终止(TERMINATED)：表示该线程已经执行完毕

- 线程池

	- 参数

		- 核心池大小
		- 线程池最大值

			- 缓冲队列排满了后扩张线程池到最大值

		- 空闲时间

			- 池中线程空闲的最大时间，达到该时间，池中线程销毁，直到线程数=核心池大小；如果allowCoreThreadTimeOut=true，那么线程数会销毁到0

		- 空闲时间单位
		- 任务队列

			- ArrayBlockingQueue

				- 定容量数组FIFO阻塞队列，put，take方法阻塞

					- 使用重入锁

			- LinkedBlockingQueue

				- 单链表FIFO阻塞队列，put，take方法阻塞，默认大小Integer.MAX_VALUE

					- 使用重入锁

			- PriorityBlockingQueue

				- 优先级阻塞队列，底层由二叉堆实现，优先级高的先出，可定制优先级

					- 使用重入锁

			- SynchronousQueue

				- 可视为容量为1的ArrayBlockingQueue

					- 使用CAS

		- 线程工厂

			- 可设置名字、是否为守护线程、优先级

		- 拒绝策略

			- 使用场景

				- 线程数达到最大线程池大小，并且阻塞队列已满的情况下继续提交任务
				- 在线程池中还有任务执行的情况下执行了shutdown，在等待未执行任务执行完的过程中继续提交任务

			- 具体策略

				- 中断策略AbortPolicy

					- 抛出RejectedExecutionException，后面的就进不去也不再管了

				- 调用者执行策略CallerRunsPolicy

					- 给调用者去执行，多了可能导致性能低下

				- 丢弃最老策略DiscardOldestPolicy

					- 依次丢弃池中和等待队列中为执行的线程

				- 丢弃策略DiscardPolicy

					- 空方法，效果和中断一样，甚至不会抛出RejectedExecutionException

	- 1.8前不建议使用Excutors创建，因为很多预设线程池大小为最大，会导致OOM；1.8后已修复
	- 线程进入线程池的流程

		- 任务进来时线程数达到核心池大小

			- 阻塞队列已满

				- 判断运行线程数是否达到最大线程数

					- 达到

						- 执行拒绝策略

					- 未达到

						- 创建新线程到最大线程数

			- 阻塞队列未满

				- 进行排队

		- 任务进来时线程数未达到核心池大小

			- 创建新线程

	- 线程获取任务与销毁

		- 如果allowCoreThreadTimeOut==false&&线程数小于等于核心池大小，那么使用workQueue.take()，该方法一直阻塞，不会超时
		- 如果allowCoreThreadTimeOut==true||线程数大于核心池大小，那么使用workQueue.poll(keepAliveTime,TimeUnit)，超时后会被works.remove(work)销毁

- ThreadLocal

	- 原理

		- 线程私有，内部有一个基于WeakReference的ThreadLocalMap，key是threadlocal实例，value是存进来的值

	- 内存泄漏

		- 原理

			- threadLocalMap的key是弱引用，如果弱引用的key被回收，但是value是个强引用

		- 例子

			- 在线程池内部使用new Threadlocal，而且没有调用remove方法，会导致内存泄漏

		- 改进方法

			- 使用全局static的Threadlocal
			- 使用完threadlocal后调用remove方法

### 引用

- StrongReference

	- 抛出OOM也不会强制回收

- SoftReference

	- 内存不足时回收

- WeakReference

	- GC时回收

- PhatomReference

	- 任何时候都可能

### 零拷贝

- 原理

	- 使用linux下的mmap API，对内核缓冲区的内存和用户缓冲区的内存进行映射

- 相关方法类

	- MappedByteBuffer=RandomAccessFile.getChannel().map(mapMode,startIndex,size)
	- FileChannel=FileInputStream.getChannel()
	- FileChannel.transferTo

- 需要注意

	- MappedByteBuffer最好手动释放，使用sun.misc.Cleaner释放

		- Cleaner cleaner =((DirectBuffer)mappedByteBuffer).cleaner();
cleaner.clean();

### 新特性

- Stream流

	- 并发流

		- ForkJoin框架

	- 创建流

		- Stream.of(T ..values)

			- 不指定范性类型时可以放入任意元素，与其他范性应用相同

		- Stream.iterate(final T seed, final UnaryOperator<T> f)

			- 首次返回seed，后续返回seed=f.apply(seed)

		- Stream.generate(Supplier<T> s)

			- 将自定义操作作为Supplier的函数式接口的get实现，实现元素的创建

		- Stream.builder()

			- 应用builder设计模式，可以通过add方法灵活地加入元素

		- Collection#stream()

			- 封装了对Collection进行迭代的方式获取元素

		- Stream.concat(Stream<? extends T> a, Stream<? extends T> b)

			- 连接两个流

	- 中间操作

		- 有状态

			- distinct
			- sorted
			- limit
			- skip

		- 无状态

			- filter
			- map
			- flatMap
			- peek
			- mapToInt
			- flatMapToInt
			- mapToLong
			- mapToDouble
			- flatMapToLong
			- flatMapToDouble

	- 终止操作

		- 短路

			- anyMatch
			- allMatch
			- noneMatch
			- findFirst
			- findAny

		- 非短路

			- count
			- forEach
			- forEachOrdered
			- toArray
			- reduce
			- collect
			- min
			- max

- 函数式接口

	- @FunctionalInterface
	- 方法唯一
	- 例子:Runnable

- Lambda

	- 它是函数式接口的实际应用

- LocalDateTime

	- LocalDateTime
	- LocalDate
	- LocalTime

- 接口增强

	- 默认方法
	- 静态方法

### 锁

- synchronized

	- 原理

		- 监视器模式

			- 锁代码块

				- 通过moniterenter和moniterexit包裹

			- 锁方法

				- 在方法上添加ACC_SYNCHRONIZED标记
				- 锁非静态方法是锁对象
				- 锁静态方法是锁类

	- 锁升级

		- 无锁
		- 偏向锁（一个线程）

			- markword记录线程ID

		- 轻量级锁（第二个线程竞争）

			- markword指向线程栈中的锁记录指针

		- 膨胀为重量级锁时会自旋锁（默认10次）

			- 自适应自旋，成功提高次数

		- 重量级锁（多个线程竞争）

			- markword指向监视器Monitor的指针

	- 锁粗化

		- 例子：在循环中不断获取锁，会把锁放在循环上，提高性能

	- 锁消除

		- StringBuffer的append方法就是同步的，如果StringBuffer对象没有逃逸，可以在server模式下使用-XX:+DoEscapeAnalysis -XX:+EliminateLocks实现逃逸分析和锁消除

	- 可重入

- AQS(AbstractQueuedSynchonizer)

	- 原理

		- volatile修饰的stats

			- CAS去修改
			- 一般情况下0未被持有，Semaphore时是允许进入数
			- 1以及以上被持有或共享，Semaphore是持有减1，释放加1

		- 双向链表队列

			- 等待锁的线程会进入队尾排队

				- 可能出现的尾分叉

					- node.prev = t;
if (compareAndSetTail(t, node)) {
          t.next = node；
          return t;
      }
					- 在进行CAS时并发失败，会出现多个节点的前置节点在同一个节点上，导致分叉，但是后期循环会解决这个问题

		- 和ObjectMonitor的owner对应的exclusiveOwnerThread

	- 模版模式
	- 实例

		- 使用Sync下的公平锁和非公平锁

			- reentrantLock

				- 独占锁

					- condition

						- Condition condition = lock.newCondition();
condition.await();
condition.signal();
condition.signalAll();

			- readWriteLock

				- 高16位读锁shareCount，低16位写锁exclusiveCount

			- Semaphore

				- 共享锁，允许共享量数量线程同时获得锁

			- countdownLatch

				- 共享锁，await()会去排队尝试获取锁，直到status在countdown的作用下归0

- CAS(CompareAndSwap)

	- 原理

		- 比较并交换，带有预期值去比较，相同才能交换，是CPU层面的原子指令

	- ABA问题

		- 加入版本号

	- 实例

		- Atomic包下的

			- AtomicInteger

				- volatile修饰的value

			- AtomicLong
			- LongAdder

				- volatile修饰的base和Cells数组

			- DoubleAdder

### volatile

- 禁止指令重排序

	- volatile写在前后加入内存屏障

		- 前一个禁止写重排序
		- 后一个禁止读重排序

	- volatile读在后面加入两个内存屏障

		- 禁止后面的读写重排序

- 内存可见性

	- 线程内的变量是存放在工作内存中的，volatile在操作时会清空工作内存，直接读主内存

### 幂等性

- 场景以及对应方案

	- 更新操作

		- 参考CAS方案。增加状态机，根据id判断状态是否符合流程

	- 新增操作

		- 增加操作token，做在redis类似的缓存中判断是否重复操作

	- 删除操作

		- 在客户端也可以视为天然幂等。业务上可以判断是否已经删除，返回视情况做不同处理

### 序列化

- 作用

	- 存储到磁盘
	- 传输于不同媒体间

- 使用

	- 实现Serializable

		- 不用自己再实现其他

	- 实现Externalizable

		- 需要自己再实现readExternal()、writeExternal()

- SerializableUID

	- 手动设置

		- 可以决定类是否能反序列化

			- 新的字段设置为初始值，删除了的字段不设置

	- 不设置

		- 会根据类名、成员变量等属性自动生成

- trasiant

	- 该修饰符修饰的成员不进行序列化

- 其他

	- 静态变量不进行序列化

## 分布式链路追踪

### OpenTracing框架

- 模块

	- Trace

		- 一个业务流

	- Span

		- 业务流下的子模块

			- 操作名称
			- 起始时间
			- 结束时间
			- 一组 KV 值，作为阶段的标签（Span Tags）
			- 阶段日志（Span Logs）
			- 阶段上下文（SpanContext），其中包含 Trace ID 和 Span ID
			- 引用关系（References）

### Jaeger

- Jaeger 是一个很经典的架构，由客户端主动发送链路信息到 Agent，Agent 上报给 Collector，再经由队列，最终落地到存储。再由另外的可视化管理后台进行查看和分析

### skywalking

### pinpoint

### Zipkin

### Elastic APM

### 鹰眼

## 限流算法

### 计数

- 描述

	- 固定请求次数

- 缺点

	- 固定
	- 可能短时间大量请求涌入

### 固定窗口

- 描述

	- 请求次数小于阈值，允许访问并且计数器 +1；
	- 请求次数大于阈值，拒绝访问；
	- 这个时间窗口过了之后，计数器清零

- 缺点

	- 在清零前后堆积请求，可能超过处理能力

### 滑动窗口

- 描述

	- 记录每次请求的时间
	- 统计每次请求的时间 至 往前推1秒这个时间窗口内请求数，并且 1 秒前的数据可以删除。
	- 统计的请求数小于阈值就记录这个请求的时间，并允许通过，反之拒绝

- 缺点

	- 可能不够平滑

### 漏桶

- 描述

	- 请求来了放入桶中
	- 桶内请求量满了拒绝请求
	- 服务定速从桶内拿请求处理

- 缺点

	- 流量大不大都很平均

### 令牌桶

- 描述

	- 定速的往桶内放入令牌
	- 令牌数量超过桶的限制，丢弃
	- 请求来了先向桶内索要令牌，索要成功则通过被处理，反之拒绝

- 缺点

	- 初始化时没令牌

## 并发与并行

### 基础

- 一个CPU一个时刻只能执行一个任务

### 区别

- 并发

	- 多个线程竞争同一个CPU，在于同一个CPU进行任务切换，也就是在一个时间线上进行时间分片，分给不同线程去执行，广义的同时在进行两个任务

- 并行

	- 两个CPU狭义的在同一个时刻进行着两个任务

## DDD领域模型

### 失血模型

- DO只有属性的get set方法的纯数据类，所有的业务逻辑完全由Service层来完成的，由于没有dao，Service直接操作数据库，进行数据持久化

	- service：肿胀
model：只包含getter setter
dao：没有

### 贫血模型

- DO包含了不依赖于持久化的原子领域逻辑，而组合逻辑在Service层

	- service：组合服务
model：除了getter setter，还包含原子服务
dao：负责持久化

### 充血模型

- 绝大多业务逻辑都应该被放在domain object里面，包括持久化逻辑，而Service层是很薄的一层，仅仅封装事务和少量逻辑，不和DAO层打交道

	- service：组合服务
model：除了getter setter，还包含原子服务和持久化逻辑
dao：没有

### 胀血模型

- 取消了Service层，只剩下domain object和DAO两层，在domain object的domain logic上面封装事务

	- service：没有
model：负责出持久化外的所有功能
dao：持久化

## 死锁产生的原因

### 互斥条件

- 公共资源只能同时被一个线程使用

### 请求和保持条件

- 线程使用一个资源的同时又去申请了其他资源，对自己的资源保持，对其他的资源请求

### 不可抢占资源

- 资源被占用，未使用完千，不可被抢占，只能自己释放

### 循环等待序列

- 对锁的请求等待序列形成循环

## 静态资源缓存刷新

### 在资源文件上加入版本号

### 强制缓存

- 情况

	- 不存在该缓存结果和缓存标识，强制缓存失效

		- 直接向服务器发起请求（跟第一次发起请求一致）

	- 存在该缓存结果和缓存标识，但该结果已失效，强制缓存失效

		- 使用协商缓存

	- 存在该缓存结果和缓存标识，且该结果尚未失效，强制缓存生效

		- 直接返回该结果

- 缓存规则

	- 页面使用meta元素或者Response的Header中

		- Cache-control

			- HTTP/1.1，目前主流。优先级更高

				- public：所有内容都将被缓存（客户端和代理服务器都可缓存）
				-             private：所有内容只有客户端可以缓存，Cache-Control的默认取值
				-             no-cache：客户端缓存内容，但是是否使用缓存则需要经过协商缓存来验证决定
				-             no-store：所有内容都不会被缓存，即不使用强制缓存，也不使用协商缓存
				-             max-age=xxx (xxx is numeric)：缓存内容将在xxx秒后失效

		- Expires

			- HTTP/1.0

### 协商缓存

- 情况

	- 携带缓存标识访问服务器，协商缓存生效，返回304，资源无更新
	- 携带缓存标识访问服务器，协商缓存失效，返回200和请求结果结果

- 协商缓存的标识

	- Last-Modified / If-Modified-Since

		- Last-Modified是服务器响应请求时，返回该资源文件在服务器最后被修改的时间
		- If-Modified-Since则是客户端再次发起该请求时，携带上次请求返回的Last-Modified值

	- Etag / If-None-Match（优先级更高）

		- Etag是服务器响应请求时，返回当前资源文件的一个唯一标识(由服务器生成)
		- If-None-Match是客户端再次发起该请求时，携带上次请求返回的唯一标识Etag值
