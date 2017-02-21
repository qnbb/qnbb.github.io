---
layout: post
title: Spring AOP
categories: Spring
description: Spring, AOP
keywords: Spring, AOP
---

## 是什么？
Spring AOP，面向切面编程，通过预编译方式和运行期动态代理（两种实现方式）实现程序功能的统一维护的一种技术。

- 相关概念
切面（Aspect）：一个关注点（逻辑功能）的模块化，这个关注点可能会涉及多个对象
切点（PointCut）：AOP执行时的断点，会有一个切入点关联式execution()
连接点（JoinPoint）：所有切点的统称
通知（Advice）：围绕切点的特定执行动作
  - 前置通知（Before advice）
  - 返回后通知（After returing advice）
  - 抛出异常后通知（After throwing advice）
  - 后通知（After(finally)advice）
  - 环绕通知（Around Advice）

  引入（Introduction）：不修改代码的前提，为类添加新的方法和属性
目标对象（Target Object）：AOP动态生成的对象
织入（Weaving）：动态生成对象

## 为什么？
- Spring AOP，主要是为了解决“横切关注点”（分布于应用多处的相同功能，与业务逻辑相分离）。
- 主要应用场景：事务、权限、缓存、懒加载、安全、记录跟踪 ...

## 怎么做？
Spring所有的切面和通知器都必须放在一个<aop:config>内（可以配置包含多个<aop:config>元素），每一个<aop:config>可以包含pointcut，advisor，aspect元素（他们必须按这个顺序进行声明）

- Code

- Spring AOP实现方式：
  - 预编译AspectJ，事先生成class文件，
  - 动态代理对象生成，根据是否使用接口，自动选择JDK动态代理（接口）或者cglib（类）生成对象