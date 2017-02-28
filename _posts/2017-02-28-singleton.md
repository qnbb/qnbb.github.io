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


