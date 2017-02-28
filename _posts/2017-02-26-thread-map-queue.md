---
layout: post
title: 同步容器 & 并发容器
categories: [Java, Thread, 容器]
description: 同步容器 & 并发容器
keywords: 同步容器, 并发容器, Thread, HashMap
---

### 同步容器

同步容器可以简单地理解为通过synchronized来实现同步的容器，如果有多个线程调用同步容器的方法，它们将会串行执行。

同步容器将它们的状态封装起来，并对每一个公有方法进行同步。主要包括：
- Vector
- Stack
- HashTable
- Collections.synchronized方法生成，例如： 
- Collectinons.synchronizedList() 
- Collections.synchronizedSet() 
- Collections.synchronizedMap() 
- Collections.synchronizedSortedSet() 
- Collections.synchronizedSortedMap() 

其中Vector（同步的ArrayList）和Stack（继承自Vector，先进后出）、HashTable（继承自Dictionary，实现了Map接口）是比较老的容器，Thinking in Java中明确指出，这些容器现在仍然存在于JDK中是为了向以前老版本的程序兼容，在新的程序中不应该在使用。Collections的方法时将非同步的容器包裹生成对应的同步容器。
同步容器在单线程的环境下能够保证线程安全，但是通过synchronized同步方法将访问操作串行化，导致并发环境下效率低下。而且同步容器在多线程环境下的复合操作（迭代、条件运算如没有则添加等）是非线程安全，需要客户端代码来实现加锁。

### 并发容器

由上面的分析我们知道，同步容器并不能保证多线程安全，而并发容器是针对多个线程并发访问而设计的，在jdk5.0引入了concurrent包，其中提供了很多并发容器，极大的提升同步容器类的性能。

#### ConcurrentHashMap

- 对应的非并发容器：HashMap 
- 目标：代替Hashtable、synchronizedMap，支持复合操作 
- 原理：JDK6中采用一种更加细粒度的加锁机制Segment“分段锁”，**减少粗粒度**，JDK8中采用CAS无锁算法
- 更多介绍，请参考 [ConcurrentHashMap总结](https://my.oschina.net/hosee/blog/675884)，[ConcurrentHashMap](http://链接网址)，[ConcurrentHashMap原理分析](https://my.oschina.net/hosee/blog/639352)

#### CopyOnWriteArrayList

- 对应的非并发容器：ArrayList 
- 目标：代替Vector、synchronizedList 
- 原理：利用高并发往往是读多写少的特性，对读操作不加锁，对写操作，先复制一份新的集合，在新的集合上面修改，然后将新集合赋值给旧的引用，并通过volatile 保证其可见性，当然写操作的锁是必不可少的了。
1. 读：不变性（array数组内部不会发生修改）
2. 写：重入锁

#### CopyOnWriteArraySet

- 对应的费并发容器：HashSet 
- 目标：代替synchronizedSet 
- 原理：基于CopyOnWriteArrayList实现，其唯一的不同是在add时调用的是CopyOnWriteArrayList的addIfAbsent方法，其遍历当前Object数组，如Object数组中已有了当前元素，则直接返回，如果没有则放入Object数组的尾部，并返回。

#### ConcurrentSkipListMap

对应的非并发容器：TreeMap 
目标：代替synchronizedSortedMap(TreeMap) 
原理：**Skip list（跳表）**是一种可以代替平衡树的数据结构，默认是按照Key值升序的。Skip list让已排序的数据分布在多层链表中，以0-1随机数决定一个数据的向上攀升与否，通过”空间来换取时间”的一个算法。ConcurrentSkipListMap提供了一种线程安全的并发访问的排序映射表。内部是SkipList（跳表）结构实现，在理论上能够在O（log（n））时间内完成查找、插入、删除操作。

#### ConcurrentSkipListSet

对应的非并发容器：TreeSet 
目标：代替synchronizedSortedSet 
原理：内部基于ConcurrentSkipListMap实现

#### ConcurrentLinkedQueue

不会阻塞的队列

对应的非并发容器：Queue 
原理：CAS，基于链表实现的FIFO队列（LinkedList的并发版本）

#### BlockingQueue：数据共享通道

比如，线程A希望线程B发一个消息，用什么方式告知线程B比较合理的呢？用BlockingQueue最为合理。

BlockingQueue是一个接口，并非一个具体的实现。 

BlockingQueue之所以适合做数据共享的通道，其关键还在于Blocking上。Blocking是阻塞的意思，当服务线程（服务线程指不断获取队列中的消息，进行处理的线程）处理完成队列中的消息后，他如何得知下一条消息何时到来呢？一种傻瓜式的方案是让这个线程按照一定的时间间隔不停的循环和监控这个队列。这是可行的一种方案，但显然造成了不必要的资源浪费，而循环周期也难以确定。而BlockingQueue很好的解决了这一个问题。它会让服务线程在队列为空是时，进行等待，当有新的消息进入队列后，自动将线程唤醒。

特点：拓展了Queue，增加了可阻塞的插入和获取等操作 
原理：通过ReentrantLock实现线程安全，通过Condition实现阻塞和唤醒 
实现类： 
1. LinkedBlockingQueue：基于链表实现的可阻塞的FIFO队列 
2. ArrayBlockingQueue：基于数组实现的可阻塞的FIFO队列 
3. PriorityBlockingQueue：按优先级排序的队列