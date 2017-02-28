---
layout: post
title: Singleton
categories: [DesignPattern]
description: Singleton
keywords: DesignPattern, Singleton
---

### 饿汉模式

- 优点：线程安全
- 缺点：要使用类中其他属性时，就会发生唯一一次的类初始化，创建instance（创建时间不可控）

```
/**
 * Created by mh on 17-2-28.
 */
public class SingletonHungry {
	//static status = 1;
	private SingletonHungry() {
		System.out.println("i am created");
	}
	private static SingletonHungry instance = new SingletonHungry();
	public static SingletonHungry getInstance() {
		return instance;
	}
}
```

### 饱汉模式

- 优点：第一次引用时，创建instance
- 缺点：要添加synchronized 同步才能线程安全

```
/**
 * Created by mh on 17-2-28.
 */
public class SingletonFull {
	private SingletonFull () {
		System.out.println("i am created");
	}
	private static SingletonFull instance = null;
	//要加锁
	public static synchronized SingletonFull getInstance() {
		if(instance != null) {
			return new SingletonFull();
		}
		return null;
	}
}

```


### 综合以上两种

- 优点：第一次引用时，创建instance；线程安全
- 原因：巧妙的使用内部类和类的初始化方式

```
/**
 * Created by mh on 17-2-28.
 */
public class SingletonHungry {
	//static status = 1;
	private SingletonHungry() {
		System.out.println("i am created");
	}
	private static SingletonHungry instance = new SingletonHungry();
	public static SingletonHungry getInstance() {
		return instance;
	}
}

```

### 小结

单例模式确保一个类在应用中只有一个实例。

#### 使用场景


1. spring 的 @Scope("singleton")

```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {  
  Object singletonObject = this.singletonObjects.get(beanName);  
  if (singletonObject == null) {  
   synchronized (this.singletonObjects) {  
    singletonObject = this.earlySingletonObjects.get(beanName);  
    if (singletonObject == null && allowEarlyReference) {  
     ObjectFactory singletonFactory = this.singletonFactories.get(beanName);  
     if (singletonFactory != null) {  
      singletonObject = singletonFactory.getObject();  
      this.earlySingletonObjects.put(beanName, singletonObject);  
      this.singletonFactories.remove(beanName);  
     }  
    }  
   }  
  }  
  return (singletonObject != NULL_OBJECT ? singletonObject : null);  
 }  
```

spring依赖注入时，使用了双重判断加锁的单例模式，首先从缓存中获取bean实例，如果为null，对缓存map加锁，然后再从缓存中获取bean，如果继续为null，就创建一个bean。这样双重判断，能够避免在加锁的瞬间，有其他依赖注入引发bean实例的创建，从而造成重复创建的结果。

2. AOP的切点定义中

3. 数据库连接池的设计一般也是采用单例模式，因为数据库连接是一种数据库资源。数据库软件系统中使用数据库连接池，主要是节省打开或者关闭数据库连接所引起的效率损耗，这种效率上的损耗还是非常昂贵的，因为何用单例模式来维护，就可以大大降低这种损耗。



