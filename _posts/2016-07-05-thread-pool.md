---
layout: post
title: 线程池
categories: [Java, Thread]
description: some word here
keywords: 线程池
---

### 常见的有4种线程池：
CachedThreadPool:创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。线程池为无限大，当执行第二个任务时第一个任务已经完成，会复用执行第一个任务的线程，而不用每次新建线程。
FixedThreadPool:可控制线程最大并发数，超出的线程会在队列中等待。
SingleThreadPool:单个的FixedThreadPool
ScheduledThreadPool:创建一个定长线程池，支持定时及周期性任务执行。

JDK1.5中引入了强大的concurrent包，其中最常用的莫过了线程池的实现ThreadPoolExecutor，它给我们带来了极大的方便，但同时，对于该线程池不恰当的设置也可能使其效率并不能达到预期的效果，甚至仅相当于或低于单线程的效率。

### 参数
ThreadPoolExecutor类可设置的参数主要有：
corePoolSize：
核心线程数，核心线程会一直存活，即使没有任务需要处理。当线程数小于核心线程数时，即使现有的线程空闲，线程池也会优先创建新线程来处理任务，而不是直接交给现有的线程处理。
核心线程在allowCoreThreadTimeout被设置为true时会超时退出，默认情况下不会退出。

maxPoolSize：
当线程数大于或等于核心线程，且任务队列已满时，线程池会创建新的线程，直到线程数量达到maxPoolSize。如果线程数已等于maxPoolSize，且任务队列已满，则已超出线程池的处理能力，线程池会拒绝处理任务而抛出异常。

keepAliveTime：
当线程空闲时间达到keepAliveTime，该线程会退出，直到线程数量等于corePoolSize。如果allowCoreThreadTimeout设置为true，则所有线程均会退出直到线程数量为0。

allowCoreThreadTimeout：
是否允许核心线程空闲退出，默认值为false。

queueCapacity：
任务队列容量。从maxPoolSize的描述上可以看出，任务队列的容量会影响到线程的变化，因此任务队列的长度也需要恰当的设置。

线程池按以下行为执行任务：
1. 当线程数小于核心线程数时，创建线程。
2. 当线程数大于等于核心线程数，且任务队列未满时，将任务放入任务队列。
3. 当线程数大于等于核心线程数，且任务队列已满
若线程数小于最大线程数，创建线程
若线程数等于最大线程数，抛出异常，拒绝任务

### 系统负载
参数的设置跟系统的负载有直接的关系，下面为系统负载的相关参数：
tasks：每秒需要处理的最大任务数量
tasktime：处理第个任务所需要的时间
responsetime：系统允许任务最大的响应时间，比如每个任务的响应时间不得超过2秒。

### 参数设置
corePoolSize:
每个任务需要tasktime秒处理，则每个线程每钞可处理1/tasktime个任务。系统每秒有tasks个任务需要处理，则需要的线程数为：tasks/(1/tasktime)，即tasks*tasktime个线程数。假设系统每秒任务数为100~1000，每个任务耗时0.1秒，则需要100*0.1至1000*0.1，即10~100个线程。那么corePoolSize应该设置为大于10，具体数字最好根据8020原则，即80%情况下系统每秒任务数，若系统80%的情况下第秒任务数小于200，最多时为1000，则corePoolSize可设置为20。

queueCapacity:
任务队列的长度要根据核心线程数，以及系统对任务响应时间的要求有关。队列长度可以设置为(corePoolSize/tasktime)*responsetime： (20/0.1)*2=400，即队列长度可设置为400。
队列长度设置过大，会导致任务响应时间过长，切忌以下写法：
```
LinkedBlockingQueue queue = new LinkedBlockingQueue();
```
这实际上是将队列长度设置为Integer.MAX_VALUE，将会导致线程数量永远为corePoolSize，再也不会增加，当任务数量陡增时，任务响应时间也将随之陡增。

maxPoolSize:
当系统负载达到最大值时，核心线程数已无法按时处理完所有任务，这时就需要增加线程。每秒200个任务需要20个线程，那么当每秒达到1000个任务时，则需要(1000-queueCapacity)*(20/200)，即60个线程，可将maxPoolSize设置为60。

keepAliveTime:
线程数量只增加不减少也不行。当负载降低时，可减少线程数量，如果一个线程空闲时间达到keepAliveTiime，该线程就退出。默认情况下线程池最少会保持corePoolSize个线程。

allowCoreThreadTimeout:
默认情况下核心线程不会退出，可通过将该参数设置为true，让核心线程也退出。
以上关于线程数量的计算并没有考虑CPU的情况。若结合CPU的情况，比如，当线程数量达到50时，CPU达到100%，则将maxPoolSize设置为60也不合适，此时若系统负载长时间维持在每秒1000个任务，则超出线程池处理能力，应设法降低每个任务的处理时间(tasktime)。

### 拒绝策略
RejectedExecutionHandler接口提供了对于拒绝任务的处理的自定方法的机会。在ThreadPoolExecutor中已经默认包含了4中策略，因为源码非常简单，这里直接贴出来。

- CallerRunsPolicy：线程调用运行该任务的 execute 本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  if (!e.isShutdown()) {
  r.run();
  }
  }
```
这个策略显然不想放弃执行任务。但是由于池中已经没有任何资源了，那么就直接使用调用该execute的线程本身来执行。

- AbortPolicy：处理程序遭到拒绝将抛出运行时 RejectedExecutionException

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  throw new RejectedExecutionException();
  }
```
 这种**策略直接抛出异常**，丢弃任务。

- DiscardPolicy：不能执行的任务将被删除

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  }
```
 这种策略和AbortPolicy几乎一样，也是丢弃任务，只不过他不抛出异常。

- DiscardOldestPolicy：如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  if (!e.isShutdown()) {
  e.getQueue().poll();
  e.execute(r);
  }
  }
```
该策略就稍微复杂一些，在pool没有关闭的前提下首先丢掉缓存在队列中的最早的任务，然后重新尝试运行该任务。这个策略需要适当小心。
设想:如果其他线程都还在运行，那么新来任务踢掉旧任务，缓存在queue中，再来一个任务又会踢掉queue中最老任务。
