---
layout: post
title: 线程池
categories: [Java, Thread]
description: some word here
keywords: 线程池
---

### 常见的有4种线程池：

####  CachedThreadPool:

创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。线程池为无限大，当执行第二个任务时第一个任务已经完成，会复用执行第一个任务的线程，而不用每次新建线程。

#### FixedThreadPool:固定大小的线程池

可控制线程最大并发数，超出的线程会在队列中等待。

```
package threadpool;

import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * Created by mh on 17-2-25.
 */
public class NewFixThreadpoolDemo {
	public static class Mytask implements Runnable {
		@Override
		public void run() {
			System.out.println(System.currentTimeMillis() + ":Thread ID:"
				+ Thread.currentThread().getId());
			try {
				Thread.sleep(2000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main (String[] args) {
		Mytask task = new Mytask();
		ExecutorService es = Executors.newFixedThreadPool(5);
		for(int i = 0;i < 10;i ++) {
			es.submit(task);
		}
	}
}
```
执行以上代码，得到类似一下输出：
```
1487991900426:Thread ID:9
1487991900428:Thread ID:10
1487991900428:Thread ID:11
1487991900431:Thread ID:13
1487991900437:Thread ID:12
1487991902428:Thread ID:9
1487991902428:Thread ID:10
1487991902429:Thread ID:11
1487991902431:Thread ID:13
1487991902437:Thread ID:12
```
说明这10个任务是分成两个批次的，符合一个只有5个线程池的行为

#### SingleThreadPool:单个的FixedThreadPool

#### cheduledThreadPool:计划任务
创建一个定长线程池，支持定时及周期性任务执行。它返回一个ScheduleExecutorService对象，可以根据时间需要对线程进行调度。


JDK1.5中引入了强大的concurrent包，其中最常用的莫过了线程池的实现ThreadPoolExecutor，它给我们带来了极大的方便，但同时，对于该线程池不恰当的设置也可能使其效率并不能达到预期的效果，甚至仅相当于或低于单线程的效率。

### 刨根问底：核心线程池的内部实现

对于以上核心的几个线程池，虽然看起来有着完全不同的功能特点，但是内部实现均使用 ThreadPoolExecutor，来看三个线程池的实现方式：

```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));

    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    }
```

由以上线程池的实现代码可以看到，他们都只是 ThreadPoolExecutor 类的封装。来看下 ThreadPoolExecutor 最重要的构造函数：

```
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

其中，
1. int corePoolSize, 指定线程池的线程数量
2. int maximumPoolSize, 制定线程池中最大的线程数量
3. long keepAliveTime, 当线程池中线程数量超过 corePoolSize 时，多余的线程的存活时间
4. TimeUnit unit, keepAliveTime 的时间单位
5. BlockingQueue<Runnable> workQueue, 任务队列，被提交尚未执行的任务
6. ThreadFactory threadFactory,  **用于创建线程的线程工厂，一般用默认的**
7. RejectedExecutionHandler handler, 拒绝策略

### workQueue

- 直接提交的队列
SynchronousQueue实现，是一个特殊的BlockingQueue。没有容量，每一个插入操作，都要等待一个相应的删除操作，相反，每一个删除操作都要等待对应的插入操作。提交的任务不会被真实的保存，而是直接提交给线程执行。用在 cacheThreadPoolExecutor。

- 有界的任务队列
ArrayBlockingQueue(int capacity) 实现。
**线程数量** < corePoolSize，直接新建线程；
corePoolSize < **线程数量** < corePoolSize + 队列大小，放入队列；
corePoolSize + 队列大小 < **线程数量** < maximumPoolSize，新建线程
maximumPoolSize < **线程数量**，执行拒绝策略

- 无界的任务队列
LinkedBlockingQueue( ) 实现。
**线程数量** < corePoolSize，直接新建线程；
corePoolSize < **线程数量** < corePoolSize + 队列大小，放入队列；
corePoolSize + 队列大小 < **线程数量**，执行拒绝策略

- 优先的任务队列
PriorityBlockingQueue实现，可以控制任务的执行先后顺序。是一个特殊的无界队列。

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

#### CallerRunsPolicy

线程调用运行该任务的 execute 本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  if (!e.isShutdown()) {
  r.run();
  }
  }
```
这个策略显然不想放弃执行任务。但是由于池中已经没有任何资源了，那么就直接使用调用该execute的线程本身来执行。

#### AbortPolicy

处理程序遭到拒绝将抛出运行时 RejectedExecutionException

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  throw new RejectedExecutionException();
  }
```
 这种**策略直接抛出异常**，丢弃任务。

#### DiscardPolicy
不能执行的任务将被删除

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  }
```
 这种策略和AbortPolicy几乎一样，也是丢弃任务，只不过他不抛出异常。

#### DiscardOldestPolicy
如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）

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
