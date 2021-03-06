# Redis

#### 数据类型

| 数据类型    | 特点                                                 | 应用场景                           |
| :---------- | ---------------------------------------------------- | ---------------------------------- |
| string      | value可以是 str/int，二进制安全                      | 缓存、计数                         |
| hash        | 可方便的操作单个field                                | 能很好的模拟session的效果          |
| list        | 按插入顺序排序的字符串链表，双向链表实现，可在头尾插 | 消息队列、lrange分页功能，最近新闻 |
| set         | 不重复的集合                                         | 去重、交并差集、共同好友列表       |
| zset        | 多了一个权重参数Score，集合中元素能按照Score进行排列 | 排行榜、计数器                     |
| Stream      | 参考kafka的消费组模型                                | Redis对消息队列的完善实现          |
| HyperLogLog | 只计算基数，不储存输入元素本身                       | 记录网站每天的独立IP访问数量       |

#### redis、memcache区别

|           | Redis             | Memcache                                            |
| --------- | ----------------- | --------------------------------------------------- |
| 持久化    | RDB、AOF          | 不支持                                              |
| 数据类型  | 5种               | string                                              |
| 线程模型  | 单线程+IO多路复用 | 多线程+锁，主线程监听，work子进程接受请求，有锁冲突 |
| value大小 | 不限制            | 限制最大1M                                          |
|           |                   |                                                     |

#### redis为什么这么快

```
1、完全基于内存，全程使用kv结构，读取速度快
2、采用单线程，避免了不必要的上下文切换和竞争条件，不用考虑锁的问题
3、使用多路I/O复用模型，epool非阻塞IO；
4、纯C实现，更接近底层
```

#### 持久化实现

RDB做镜像全量持久化，AOF做增量持久化。因为RDB会耗费时间，不够实时，在停机的时候会导致大量丢失数据，所以需要AOF来配合使用。在redis实例重启时，会使用RDB持久化文件重新构建内存，再使用AOF重放近期的操作指令来实现完整恢复重启之前的状态。

- RDB。在一个特定的间隔保存那个时间点的一个数据快照（SAVE、BGSAVE）

  fork和cow原理，fork是指redis通过创建子进程来进行RDB操作，cow指的是**copy on write**，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来。

  ```
  dbfilename dump.rdb		# RDB文件名，默认为dump.rdb。
  
  save 900 1 			#900秒后至少1个key有变动
  save 300 10 		#300秒后至少10个key有变动
  save 60 10000 		#60秒后至少10000个key有变动
  ```

- AOF。记录每一个收到的写操作。

  ```
  appendonly yes						# 是否开启aof
  appendfilename "appendonly.aof"				# 文件名称
  appendfsync everysec 				# 同步方式，推荐每秒同步一次
  ```

  同步方式有三种模式：`always`：把每个写命令都立即同步到aof，很慢，但是很安全；`everysec`：每秒同步一次，是折中方案；`no`：redis不处理交给OS来处理，非常快，但是也最不安全

#### 缓存一致性

强一致性的场景不应该用缓存，应该通过增加mysql性能来实现。

```
1.先删缓存，写数据库成功后再删一次缓存。
2.读取mysql的binlog后分析 ，利用消息队列,推送更新各平台的redis缓存数据，参考`canal-python`。
```

#### 穿透、击穿、雪崩

|      | 原因                                              | 解决方案                                |
| ---- | ------------------------------------------------- | --------------------------------------- |
| 穿透 | 不断请求一个不存在的数据导致mysql频繁查询，负载大 | 参数校验、缓存空值、ip限流、Bloom过滤器 |
| 击穿 | 热点key失效，大量请求打到mysql                    | 设置key永不过期、加分布式锁             |
| 雪崩 | 大量key同时失效                                   | 失效时间加上随机秒数                    |

#### 主从同步机制

Redis可以使用主从同步，从从同步。第一次同步时，他会发送一个**psync**命令给master，主节点做一次**bgsave**，同时将后续写操作记录到内存buffer。RDB生成后，会将文件全量同步到slave节点，slave接受完成后将RDB镜像加载到内存，再通知master将期间修改的操作记录(buffer中内容)同步到复slave节点进行重放就完成了同步过程。后续的增量数据通过AOF日志同步即可，有点类似数据库的binlog。

#### 过期策略

```
1.惰性清理。键过期了先不处理，当读/写到过期key时，直接删除掉这个过期key
2.定期清理。默认每隔100ms，对expires字典随机取key，检查是否过期，如果过期则删除
```

redis采用惰性删除+定期删除的策略。定时清理的频率和`hz(默认10)`参数有关，即CPU空闲时每秒执行10次；遍历db，随机取20个key，判断是否过期，若过期，则逐出；有5个以上key过期，则重复取key，否则遍历下一个db；在清理过程中，若达到了25%CPU时间(默认25ms)，退出此次清理过程;

#### 内存淘汰机制

当redis内存超出物理内存限制时，会和磁盘产生swap，这种情况性能极差，一般是不允许的。通过设置 `maxmemory(默认0)` 限制最大使用内存。超出限制时，根据redis提供的几种内存淘汰机制让用户自己决定如何腾出新空间以提供正常的读写服务。

```
noeviction：当内存使用超过配置的时候会返回错误，不会驱逐任何键（默认策略，不建议使用）
allkeys-lru：通过LRU算法驱逐最久没有使用的键
volatile-lru：先从设置了过期时间的键集合中驱逐最久没有使用的键（不建议使用）
allkeys-random：从所有key随机删除
volatile-random：从过期键的集合中随机驱逐（不建议使用）
volatile-ttl：从配置了过期时间的键中驱逐马上就要过期的键
volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键
allkeys-lfu：从所有键中驱逐使用频率最少的键
```

#### 集群模式

- 主从复制模式。可以进行读写分离，但是不具备自动容错和恢复功能。
-  Sentinal 哨兵模式。着眼于高可用，在master宕机时会自动将slave提升为master，继续提供服务。最少需要一主三从，所有主从节点都完整的保存了一份数据。`redis-py-cluster`
-  Cluster 着眼于扩展性，在单个redis内存不足时，使用Cluster进行分片存储。

