---
title: golang内存管理
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["golang"]
tags: ["golang"]
---

# golang内存管理
#技术/golang

程序中的数据和变量都会被分配到程序所在的虚拟内存中，内存空间包含两个重要区域：栈区（Stack）和堆区（Heap）。函数调用的参数、返回值以及局部变量大都会被分配到栈上，这部分内存会由编译器进行管理；不同编程语言使用不同的方法管理堆区的内存，C++ 等编程语言会由工程师主动申请和释放内存，Go 以及 Java 等编程语言会由工程师和编译器共同管理，堆中的对象由内存分配器分配并由垃圾收集器回收。

## golang
内存管理一般包含三个不同的组件，分别是
- 用户程序（Mutator）
- 分配器（Allocator）
- 收集器（Collector）

当用户程序申请内存时，它会通过内存分配器申请新内存，而分配器会负责从堆中初始化相应的内存区域。

### 分级分配 
线程缓存分配（Thread-Caching Malloc，TCMalloc）是用于分配内存的机制，它比 glibc 中的 malloc 还要快很多。Go 语言的内存分配器就借鉴了 TCMalloc 的设计实现高速的内存分配，它的核心理念是使用多级缓存将对象根据大小分类，并按照类别实施不同的分配策略。

**对象大小 **
Go 语言的内存分配器会根据申请分配的内存大小选择不同的处理逻辑，运行时根据对象的大小将对象分成微对象、小对象和大对象三种：

微对象 -> (0, 16B)
小对象 -> [16B, 32KB]
大对象 -> (32KB, +∞)

因为程序中的绝大多数对象的大小都在 32KB 以下，而申请的内存大小影响 Go 语言运行时分配内存的过程和开销，所以分别处理大对象和小对象有利于提高内存分配器的性能。

**多级缓存 **
内存分配器不仅会区别对待大小不同的对象，还会将内存分成不同的级别分别管理，TCMalloc 和 Go 运行时分配器都会引入
- 线程缓存（Thread Cache）
- 中心缓存（Central Cache）
- 页堆（Page Heap）
三个组件分级管理内存

线程缓存属于每一个独立的线程，它能够满足线程上绝大多数的内存分配需求，因为不涉及多线程，所以也不需要使用互斥锁来保护内存，这能够减少锁竞争带来的性能损耗。当线程缓存不能满足需求时，运行时会使用中心缓存作为补充解决小对象的内存分配，在遇到 32KB 以上的对象时，内存分配器会选择页堆直接分配大内存。

这种多层级的内存分配设计与计算机操作系统中的多级缓存有些类似，因为多数的对象都是小对象，我们可以通过线程缓存和中心缓存提供足够的内存空间，发现资源不足时从上一级组件中获取更多的内存资源。

Go 语言程序的 1.10 版本在启动时会初始化整片虚拟内存区域，分为三个区域 ：spans、bitmap 和 arena 分别预留了 512MB、16GB 以及 512GB 的内存空间，这些内存并不是真正存在的物理内存，而是虚拟内存

* spans 区域存储了指向内存管理单元  [runtime.mspan](https://draveness.me/golang/tree/runtime.mspan)  的指针，每个内存单元会管理几页的内存空间，每页大小为 8KB；
* bitmap 用于标识 arena 区域中的那些地址保存了对象，位图中的每个字节都会表示堆区中的 32 字节是否空闲；
* arena 区域是真正的堆区，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象；

### mcache
mcache是线程的私有缓存，每个工作线程都会绑定一个mcache。mcache中缓存着线程私有的mspan资源。由于mcache是每个线程私有的，所以线程在从mcache中申请mspan资源时不会产生线程间的竞争。
mcache中维护了一组（共134个，67规格*2）mspan链表，分别对应不同大小的class。根据对象是否包含指针，将对象分为noscan和scan两类分别放到不同的链表中，其中noscan代表没有指针，scan代表有指针。
当变量需要小于 32 KiB 内存时，Go 语言会从名为 mcache 中为小于 32 KiB 的需求申请内存，该缓存处理内存块大小为小于等于 32 KiB 的可分配内存的链表，名为 mspan
每个线程 M 被分配至一个处理机 P，且最多可一次处理一个 Goroutine。分配内存时，当前的 Goroutine 会使用其被分配的处理机 P 来透过 mcache 找到 span 列表中第一个可用的 object
mcache 上还有**微型分配器**，当要分配更小元素：即 <= 16B 时，会在一个 8byte 的 mspan 上分配多个的对象，这样就能更好的利用内存空间。

### mcentral
**mcentral** 维护了各个规格的 mspan。当它的下级 **mcache** 内存不足时，则会到 mcentral 这里来申请 mspan。
由于 mcentral 有各个规格类型的 mspan，因此当有不同规格的分配请求时，并不会产生并发竞争的问题。只有当同类型规格的 mspan 并发请求分配时，才会有加锁操作。
当本地缓存的 span 用完时，Go 语言会请求从 mcentral 获取一个新的 span，追加至本地链表中

mcache、mcentral都属于arena

### mheap
再当 mcentral 没有可用的内存供 span 分配时，Go 语言再透过 OS 从 heap 中申请新的内存并加入 mcentral 的链表中


## 总结
1. object size > 32K，则使用 mheap 直接分配。 
2. object size < 16 byte，不包含指针使用 mcache 的小对象分配器 tiny 直接分配；包含指针分配策略与[16 B, 32 K]类似。 
3. object size >= 16 byte && size <=32K byte 时，先使用 mcache 中对应的 size class 分配。 
4. 如果 mcache 对应的 size class 的 span 已经没有可用的块，则向 mcentral 请求。 
5. 如果 mcentral 也没有可用的块，则向 mheap 申请，并切分。 
6. 如果 mheap 也没有合适的 span，则向操作系统申请
