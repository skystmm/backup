---
title: redis笔记
date: 2020-11-22 21:33:07
tags: [reids,技术]
---


# redis基础知识
<!-- more -->
![redis思维导图.jpg](http://image.stxl117.top/redis.jpg)

## 数据结构

### 底层数据结构

#### SDS

#### 双向链表

#### 压缩列表

#### 跳跃表

#### Hash表

#### 整数数组

### 对外数据结构

string

list

map

set

sorted set

> rehash如何触发和渐进式机制
>
> 1.  rehash触发根据负载因子阈值进行触发，不同于java中hashMap ，reids默认提供两个扩容负载因子阈值分别为1和5；
>
> 1.  当负载系数达到1时， 如果当前redis server没有执行AOF重写或者RDB备份，会触发扩容
> 2.  当负载系数达到5时，此时会立即触发阔扩容
>
> 2.  rehash流程
>
> 1.  rehash时会维护两个hash桶，一个是扩容前的老hash桶，一个是扩容后的新Hash桶
> 2.  渐进式扩容方式为：
>
> 1.  设置rehashIndex为0 表示开始扩容
> 2.  当在rehash过程中，有客户端命令执行时，除了进行正常的操作外，还会将当前rehashindex对应的桶中数据迁移到新hash桶中，rehashindex 加一
> 3.  当没有命令执行时，redis服务也会使用定时任务，执行迁移过程（每次执行不超过1ms）
> 4.  当所有老hash桶中数据都迁移完成时 rehashindex设置为-1表名迁移结束
>
> 3.  缩容[待学习]

## IO模型

6.0以前 单线程多路复用

6.0以后 多线程多路复用

## 备份

### AOF

**redis中提供三种AOF的策略**：

1.  always ——每次执行完写命令后立即写入AOF文件
2.  every second ——每次执行完写命令后，先交AOF信息写到buffer中，每秒redis将buffer中的AOF信息写入到AOF日志中
3.  no —— buffer写入AOF中的操作交友操作系统执行

三种方式数据丢失情况有少到多，性能消耗由多到少

AOF的缺点：恢复时需要回放所有命令，相对恢复时间较长；由于记录每次操作，文件相对较大

由于AOF写存在对于相同Key的多次操作，相对会有重复和冗余，所以redis提供了AOF重写机制，来减小文件大小和回访时重复执行命令的问题

**AOF重写流程大致如下**：

主进程fork出 bgrewriteaof进程，并将页表复制给子进程，子进程根据页表数据进行aof重写合并

操作过程中的写入命令，主进程将写到AOF重写缓冲中，也将共享给bgrewirteaof进程，自己成两者一起进行合并，生成洗的Aof日志并落盘

触发方法：

1.  手动执行bgrewriteaof命令
2.  server周期性执行，满足以下条件会触发

*   没有BGSAVE命令（RDB持久化）/AOF持久化在执行；
*   没有BGREWRITEAOF在进行；
*   当前AOF文件大小要大于`server.aof_rewrite_min_size`（默认为1MB），或者在`redis.conf`配置了`auto-aof-rewrite-min-size`大小；
*   当前AOF文件大小和最后一次重写后的大小之间的比率等于或者等于指定的增长百分比（在配置文件设置了`auto-aof-rewrite-percentage`参数，不设置默认为100%）

### RDB

redis中提供两个命令触发RDB备份 `save` 和`bgsave` 。

`save` 命令是由主线程执行的对当前redis内存进行快照，执行时会阻塞主线程

`bgsave` 命令则是由fork出子进程进行内存快照，除了fork过程，不会阻塞主线程执行命令；同时在进行快照过程中新处理的命令通过COW方式复制副本给子进程，来保证最后的一致性

![image.png](http://image.stxl117.top/rdb.png "RDB")

同时Redis可以在配置文件中配置rdb的策略

`save 900 1`

`save 300 10`

`save 60 10000`    

来配置rdb策略，上面命令900秒内少于1次变更操作就会触发一次主线程的rdb，以此类推。总体来说是server本身根据自身繁忙程度来确定什么时间进行备份。

### 混合模式

上面记录了AOF和RDB的基本处理逻辑，会发现AOF和RDB各有各的优点，但也有各自的不足之处。例如AOF可以做到数据最少丢失，但是恢复时需要进行命令回放，速度会比较慢；而RDB则是通过快照的方式进行备份，恢复速度快，但是备份间隔期间出现宕机会丢失从上次备份到当前时间点的数据。可以看到二者有互补的地方，所有redis 4.0后支持了 RDB和AOF的混合模式保证数据可靠性

> 选择什么样的备份策略还是要看业务形态等因素

## 高可用

### 1\. 主从

**主从同步流程**

**![redisms.png](http://image.stxl117.top/ms.png "redisms.png")**

**同步方式： 全量同步（rdb），长连接命令传输，增量同步（重连后）**

> repl_backlog_buffer和replication buffer的区别
>
> repl_backlog_buffer 为了主从差异数据设计的环形缓冲队列，只要有从库链接就会产生，所有从库共享
>
> replication buffer 每个redis client的链接创建一个缓冲区，对于主从同步情况的特殊叫法，一般流程为Redis将返回数据写到该buffer中，然后再从该buffer中获取数据写入到对应的socket中返回给从库

#### 哨兵

**作用**

1.  监听

主观下线：单个哨兵监听超时

客观下线：主观下线的数量大于等于配置的quorum值

2.  选主

1.  筛选：当先在线且网络通信质量比较好的从库
2.  打分

1.  配置项中优先级最高的实例
2.  slave_replication_offset最大的
3.  runId最小的

3.  通知

**本质：****哨兵是一个特定模式的redis实例**

#### 哨兵集群

**基本信息**

通过配置主库信息，链接主库，并通过pub/sub机制获取其他哨兵服务抵制并发布自身ip信息，达到组网目的

从库信息通过使用INFO命令从主库中获得

客户端获取信息通过哨兵的pub/sub方式获取变更信息

**哨兵集群选主——类似raft**

选票大于等于配置的quorum值，且大于等于集群节点 N/2 + 1票

#### redis cluster

水平扩展

去中心化

路由寻址：crc16的slot

数据迁移： move,ask,asking

## 淘汰策略

对于redis提供的缓存淘汰策略，大致可以分为三类

1.  无缓存清理策略(noeviction) :不会进行缓存淘汰，使用本策略时，当redis内存达到阈值后，将会拒绝后续的写请求，直接返回错误
2.  对于有过期时间的Key的淘汰策略:其中包含四种淘汰策略volitile-radom,volitile-ttl,volitile-lru,volitile-lfu
3.  全局key淘汰策略:allkeys-lru,allkeys-lfu,allkeys-random

**关于lru实现和一般lru算法不同的地方**：

> LRU是对最近最不常使用的节点进行淘汰的策略，一般使用hashMap和双向链表实现

由于redis中存在海量的key,同时内存也属于稀缺资源，如果维护一个全局双向链表将会产生很大的内存开销，所以redis对lru进行优化,采用了近似LRU进行缓存淘汰，可以参考[https://zhuanlan.zhihu.com/p/34133067](https://zhuanlan.zhihu.com/p/34133067)

**LFU[待补充]**

> LFU是对最近使用频率最小的节点进行淘汰的策略，一般使用hashMap和小根堆实现

# redis使用姿势

[高并发和海量数据下的 9 个 Redis 经典案例剖析！](https://mp.weixin.qq.com/s/wT3i12HV_6ev5HSPJ9ANRg)
[咆哮位图](https://juejin.im/post/6844903859383451656)
