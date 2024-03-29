---
title: redis知识点汇总——数据存储
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["redis"]
tags: ["redis"]
---

# redis知识点汇总——数据存储
#技术/数据库/redis

## 内存淘汰策略
1. noeviction
	当内存使用超过配置的时候会返回错误，不会驱逐任何键
2. allkeys-lru
	- 加入键的时候，如果过限，首先通过LRU算法驱逐最久没有使用的键
	- Redis中的LRU与常规的LRU实现并不相同，常规LRU会准确的淘汰掉队头的元素，但是Redis的LRU并不维护队列，只是根据配置的策略要么从所有的key中随机选择N个（N可以配置）要么从所有的设置了过期时间的key中选出N个键，然后再从这N个键中选出最久没有使用的一个key进行淘汰
3. volatile-lru
	加入键的时候如果过限，首先从设置了过期时间的键集合中驱逐最久没有使用的键
4. allkeys-random
	加入键的时候如果过限，从所有key随机删除
5. volatile-random
	加入键的时候如果过限，从过期键的集合中随机驱逐
6. volatile-ttl
	从配置了过期时间的键中驱逐马上就要过期的键
7. volatile-lfu
	从所有配置了过期时间的键中驱逐使用频率最少的键
8. allkeys-lfu
	从所有键中驱逐使用频率最少的键

## redis lru的实现
Redis配置中和LRU有关的有三个：
* maxmemory: 配置Redis存储数据时指定限制的内存大小，比如100m。当缓存消耗的内存超过这个数值时, 将触发数据淘汰。该数据配置为0时，表示缓存的数据量没有限制, 即LRU功能不生效。64位的系统默认值为0，32位的系统默认内存限制为3GB
* maxmemory_policy: 触发数据淘汰后的淘汰策略
* maxmemory_samples: 随机采样的精度，也就是随即取出key的数目。该数值配置越大, 越接近于真实的LRU算法，但是数值越大，相应消耗也变高，对性能有一定影响，样本值默认为5。

Redis Object以秒为单位存储了对象新建或者更新时的
unix time。
Redis的键空间是放在一个哈希表中的，要从所有的键中选出一个最久未被访问的键，需要另外一个数据结构存储这些源信息，这显然不划算。因此算法为随机选择N个key的策略，默认是5个。

## 数据存储
一个redis可以保存多个redisDb，默认为16，可以配置

### redisDb的结构
- expires
redisDb结构的expires字典保存了数据库中所有键的过期时间
	- 键：指针，指向键空间中的某个对象
	- 值：值是一个long long类型的整数，保存了ms精度的过期时间，是一个unix时间戳
- dict
数据库键空间，保存着数据库中的所有键值对
#### 过期键的删除策略
惰性删除+定期删除
1. 所有读写数据库的redis命令都会在执行之前对输入键进行是否过期的检查，如果过期则删除
2. 当redis的周期性操作serverCron函数执行时，在规定时间内，遍历服务器中的各个数据库，每次从过期字典中随机选一部分key检查是否过期，过期则删除。如果一个db中的过期字典都被删完了，删除进度+1，指向下一个要删除的数据库
	- aof，rdb，复制功能对过期键的处理
		1. rdb
		- 生成时会检查，过期的键不会被保存到rdb中
		- 载入时：
			- 主服务器：载入时会检查，过期的键不再被载入
			- 从服务器：不论是否过期都会被载入，但是在进行主从同步时，从服务器的数据会被清空
		2. aof
			- 当过期键被惰性删除或定期删除后，程序会向aof文件追加一条DEL命令
			- aof重写时，会检查，如果过期了则不再写入新的aof文件
			- 从服务器的过期键删除动作由主服务器来控制，如果从服务器接收到客户端对过期键的请求，即使已经过期了也不会将其删除
	