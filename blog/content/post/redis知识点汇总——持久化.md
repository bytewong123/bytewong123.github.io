---
title: redis知识点汇总——持久化
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["redis"]
tags: ["redis"]
---

# redis知识点汇总——持久化
#技术/数据库/redis

## rdb
rdb是通过保存数据库中的键值对来记录数据库状态

### 指令
save：save命令会阻塞redis进程，知道rdb文件创建完为止，阻塞期间redis不能处理任何命令请求
bgsave：bgsave会派生出一个子进程，然后由子进程负责创建rdb文件，父进程可以继续处理命令请求；
> bgsave执行期间客户端发送的save/bgsave命令会被redis拒绝，防止产生竞争条件；bgrewriteaof和bgsave不能同时执行，是因为这两个命令都会创建子进程，这两个子进程同时执行大量磁盘操作，对机器的负载会比较高

bgsave过程：
1. redis在持久化时会调用glibc的函数fork出一个子进程，快照持久化完全交给子进程来处理，父进程继续处理客户端的请求。子进程刚刚产生的时候，它和父进程共享内存里的代码段和数据段。这是linux的机制，为了节约内存资源尽可能让它们共享起来。在进程分离的一瞬间，内存增长几乎没有明显的变化
2. 子进程做数据持久化，不会修改现有内存数据结构，只是对数据结构进行遍历读取，然后写盘。但父进程需要持续服务客户端的请求，然后对内存数据结构进行不间断的修改
3. fork()之后，kernel把父进程中所有的内存页的权限都设为read-only，然后子进程的地址空间指向父进程。当父子进程都只读内存时，相安无事。当其中某个进程写内存时，CPU硬件检测到内存页是read-only的，于是触发页异常中断（page-fault），陷入kernel的一个中断例程。中断例程中，kernel就会**把触发的异常的页复制一份**，于是父子进程各自持有独立的一份
4. 随着父进程修改操作的持续进行，越来越多的共享页面被分离出来，内存就会持续增长，但是也不会超过原有数据内存的2倍大小。由于redis中的冷数据占的比例较高，所有很少会有所有页面都被分离的情况。被分离的往往只有其中一部分页面
5. 子进程由于不会修改内存中的数据，它看到的内存数据在进程产生的一瞬间就不再改变。只需要遍历数据，写入磁盘即可

cow的优缺点
优点：
1. COW技术可减少**分配**和**复制**大量资源时带来的**瞬间延时**
2. COW技术可减少**不必要的资源分配**。比如fork进程时，并不是所有的页面都需要复制，父进程的**代码段和只读数据段都不被允许修改，所以无需复制**
缺点：
1. 如果在fork()之后，父进程都还需要继续进行写操作，**那么会产生大量的分页错误(页异常中断page-fault)**，这样就得不偿失

## aof
aof是通过保存redis执行的写命令来记录数据库状态
1. 命令追加
当redis的aof功能打开时，服务器执行完一个写命令之后，会以协议格式将被执行的写命令追加到aof_buf缓冲区末尾
2. 文件写入
服务器每次结束一次事件循环之前，都会调用aof.c/flushAppendOnlyFile 函数，考虑是否要将AOF缓存区的内容保存到AOF文件
	1. always：每次循环到flushAppendOnlyFile函数时，就将其刷入磁盘
	2. everysec：上次aof文件写入时间与当前超过1秒钟，则刷入文件
	3. no：不对aof文件进行同步，由操作系统决定何时同步

3. aof文件载入和数据还原
	1. 创建一个不带网络连接的伪客户端
	2. 伪客户端依次读aof文件的所有写命令并执行

4. aof重写（bgwriteaof）：为了解决aof文件体积膨胀的问题
开辟一个子进程对内存进行遍历，转换成一系列的redis操作指令，序列化到一个新的AOF日志文件中。序列化完毕后再将操作期间发生的增量AOF日志追加到这个新的AOF日志文件中，追加完毕后就立即替代旧的AOF日志文件了，瘦身工作就完成了

## 混合持久化
混合持久化同样也是通过bgrewriteaof完成的，不同的是当开启混合持久化时，fork出的子进程先将共享的内存副本全量的以RDB方式写入aof文件，然后在将重写缓冲区的增量命令以AOF方式写入到文件，写入完成后通知主进程更新统计信息，并将新的含有RDB格式和AOF格式的AOF文件替换旧的的AOF文件。简单的说：新的AOF文件前半段是RDB格式的全量数据后半段是AOF格式的增量数据
> 兼容性差，一旦开启了混合持久化，在4.0之前版本都不识别该aof文件

