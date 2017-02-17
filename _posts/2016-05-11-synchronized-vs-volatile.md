---
layout: post
title: Synchronized与Volatile可见性分析
categories: Java
description: Synchronized, Volatile
keywords: Synchronized, Volatile
---


# 1. 基础概念
**可见性**
一个线程对共享变量值的修改，能够及时地被其他线程看到。
可见性的实现方式：Java`语言层面`（不包括jdk1.5之后引入的其他高级特性）支持的可见性实现方式：

- synchronized
- volatile

**共享变量**
如果一个变量在多个线程的工作内存中都存在副本，那么这个变量就是这几个线程的共享变量。

**Java内存模型（JMM）**

- JMM描述了Java程序各种变量（线程`共享变量`）的访问规则，以及在JVM中将变量存储到内存和从内存中读取出变量的底层细节
- 所有的变量都存储在主内存中
- 每个线程都有自己独立的工作内存，里面保存改线程使用到的变量的副本（主内存中该变量的一份拷贝）

线程1 <==> 工作内存1（x的副本1） <==> 主内存（共享变量X）

线程2 <==> 工作内存2（x的副本2） <==> 主内存（共享变量X）

线程3 <==> 工作内存3（x的副本3） <==> 主内存（共享变量X）

两条规则：
1. 线程对共享内存的所有操作都必须在低级的工作内存中进行，不能直接从主内存中读写
2. 线程1无法直接访问线程2工作内存中的变量，线程之间变量值的传递需要通过主线程来完成

实现可见性！线程1对共享变量的修改要被线程2看到，必须经过以下两个步骤：
1. 把工作内存1中更新过的变量刷新到主内存中
2. 将主内存中最新的共享变量值更新到工作内存2中

**重排序**
代码书写的顺序与实际的执行顺序不同，指令重排序编译器或者处理器为了提高程序性能而做的优化

- 编译器优化的重排序（编译器优化）
- 指令级并行重排序（处理器优化）
- 内存系统的重排序（处理器优化）

**as-if-serial**
as-if-serial:无论如何重排序，程序执行的结果应该与代码顺序执行的结果一致（Java编译器、运行时和处理器都会保证Java在单线程下遵循as-if-serial语义）
`在单线程中，重排序不会带来内存可见性问题；但是多线程中程序交错执行，重排序可能会造成内存可见性问题`

**导致共享变量在线程间不可见的原因：**
1. 线程的交差执行
2. 重排序结合线程交叉执行
3. 共享变量更新后的值没有在工作内存与主内存间及时更新

**final也能实现可见性**

**为什么即使没有保证可见性措施，很多时候，共享变量在主内存和工作内存仍能及时刷新？**
一般只有在短时间内高并发的情况下才会出现更新得不到刷新，因为CPU执行会很快的刷新缓存，所以一般很难看到这种问题。

**对64位（long、double）变量的读写可能不是原子操作**
Java内存模型允许JVM将没被volatile修身的long、double64位的数据类型分为两次32位的读写操作

-------------------

# 2. synchronized实现可见性
- 同步性（原子性）
- 可见性（常常被忽略）

**JMM关于synchronized的两条规定：**

- 线程解锁前，必须将共享变量的最新值刷新到主内存中
- 线程加锁时，将清空工作内存中共享变量的值，从而使用共享变量时需要从主内存总重新读取最新的值（加锁、解锁需要同一把锁）

`线程解锁钱对共享变量的修改在下次加锁时对其他线程可见`，线程执行互斥代码的过程：
1. 获得互斥锁
2. 清空工作内存
3. 从主内存拷贝变量的最新副本到工作内存
4. 执行代码
5. 将更改后的共享变量的值刷新到主内存
6. 释放互斥锁

**不可见原因：　　　　　　　　synchronized解决方案：**
1. 线程的交差执行　　　　　　＝＝>　　　　　原子性
2. 重排序结合线程交叉执行　　＝＝>　　　　　原子性
3. 共享变量不能及时更新　　　＝＝>　　　　　可见性

---------------

# 3. volatile实现可见性
- 能保证volatile变量的可见性
- 不能保证volatile变量复合操作的原子性

**volatile如何实现内存可见性：**
深入来说：通过加入内存屏障和禁止重排序优化来实现

- 对volatile变量执行写操作时，会在写操作后加入一条store屏障指令（使缓存失效，强制让缓存和主内存同步）
- 对volatile变量执行读操作时，会在读操作前加入一条load屏障指令

人话：volatile变量在每次被线程访问时，都强迫从主内存中重读该变量的值；当变量发生改变时。又强迫线程就将最新的值刷新到主内存。这样任何时刻，不同的线程总能看到该变量的自信值

**线程写volatile过程**
1. 改变线程工作内存中volatile变量副本值
2. 将改变后的值从工作内存刷新到主内存
**线程读volatile过程**
1. 从主内存读取volatile变量的最新值，刷新到工作内存
2. 线程从工作内存中读取volatile变量副本

```
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class VolatileDemo {

private Lock lock = new ReentrantLock();
private int number = 0;

public int getNumber(){
return this.number;
}

public void increase(){
try {
Thread.sleep(100);
} catch (InterruptedException e) {
// TODO Auto-generated catch block
e.printStackTrace();
}
lock.lock();
try {
this.number++; //需要加锁，++ 非原子性
} finally {
lock.unlock();
}
  // 同lock synchronized(this) { this.number++; }

}

/**
* @param args
*/
public static void main(String[] args) {
// TODO Auto-generated method stub
final VolatileDemo volDemo = new VolatileDemo();
for(int i = 0 ; i < 500 ; i++){
new Thread(new Runnable() {

@Override
public void run() {
volDemo.increase();
}
}).start();
}

//如果还有子线程在运行，主线程就让出CPU资源
//直到所有子线程运行晚，主线程再继续往下执行
while(Thread.activeCount() > 1){
Thread.yield();
}

System.out.println("number : " + volDemo.getNumber());
}

}
```

解决number++ 自增操作的原子性：

- synchronized
- ReentrantLock（java.until.concurrent.locks包下）
- AtomicInterger原子类(vava.util.concurrent.atomic包下)

**volatile适用场合**
1. 对变量的写入操作不依赖当前值

- 不满足：number++、count = count*5等
- 满足：boolean变量、记录温度变化的变量等

2.该变量没有包含在其他变量的不变式中

- 不满足：不变式 low < up