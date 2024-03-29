---
title: redis知识点汇总——主从复制
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["redis"]
tags: ["redis"]
---

# redis知识点汇总——主从复制
#技术/数据库/redis
## 复制
客户端可以通过执行SLAVEOF命令或设置slaveof选项让一个redis实例去复制另一个实例。复制的实例称为从节点，被复制的实例被称为主节点

PSYNC
1. 完整重同步：用于处理初次复制情况
	1. 主节点执行bgsave命令，在后台生成一个rdb文件，并使用一个缓冲区记录从现在开始执行的所有写命令
	2. 主节点bgsave执行完后将生成的rdb文件发送给从节点，从节点接收并载入rdb，使自己的数据库状态与主节点执行bgsave时的状态一致
	3. 主节点将缓冲区里的所有写命令发送给从节点，从节点执行这些写命令
2. 部分重同步：用于处理断线后重连的情况，当从节点在断线后重连主节点时，主节点将断开期间执行的写命令发送给从节点
	1. 复制偏移量
		1. 主从节点会分别维护一个复制偏移量
		2. 主节点每次向从节点传播n个字节数据时，就给offset增加n
		3. 从节点每次收到主节点传播的n个字节，就将自己的offset增加n
	2. 复制积压缓冲区
		1. 固定长度的先进先出队列，默认大小1mb
		2. 缓冲区记录了每个 偏移量对应的字节值
		3. 当从节点重新连接上主节点后，发送自己的偏移量，如果主节点在这个偏移量之后的数据仍然在缓冲区中，就执行部分重同步操作
	3. 主节点的运行id
	从节点重新连上主节点后，向当前主节点发送自己保存的运行id，如果相同则说明主节点未切换过，执行部分重同步；否则说明主节点切换了，需要执行完整重同步