---
layout: post
title: 类加载机制
categories: Java
description: classloader
keywords: classloader
---
 
## 类的加载过程
JVM将类加载过程分为三个步骤：装载（Load），链接（Link）和初始化(Initialize)链接又分为三个步骤。

- 装载：查找并加载类的二进制数据；
- 链接（三步）
1. 验证：确保被加载类的正确性；
  - 文件格式验证（Class文件格式规范，如魔数0xCAFEBABE，主次版本号等）
  - 元数据验证（Java语言规范，字节码的描述信息进行语义分析，如父类、final、抽象类等基本属性合法）
  - 字节码验证（通过数据流、控制流分析，确保程序语义合法）
2. 准备：为类的静态变量分配内存，并将其初始化为默认值，常量final除外
3. 解析：常量池；
解析：把类中的符号引用转换为直接引用；
- 初始化：为类的静态变量赋予正确的初始值；
- 使用（类的生命周期）
- 卸载（类的生命周期）

那为什么我要有验证这一步骤呢？首先如果由编译器生成的class文件，它肯定是符合JVM字节码格式的，但是万一有高手自己写一个class文件，让JVM加载并运行，用于恶意用途，就不妙了，因此这个class文件要先过验证这一关，不符合的话不会让它继续执行的，也是为了安全考虑吧。

准备阶段和初始化阶段看似有点矛盾，其实是不矛盾的，如果类中有语句：private static int a = 10，它的执行过程是这样的，首先字节码文件被加载到内存后，先进行链接的验证这一步骤，验证通过后准备阶段，给a分配内存，因为变量a是static的，所以此时a等于int类型的默认初始值0，即a=0,然后到解析（后面在说），到初始化这一步骤时，才把a的真正的值10赋给a,此时a=10。

## 加载

在加载阶段，虚拟机主要需要完成以下3件事情：
1. 通过类的.class文件（类的全限定名）获得定义此类的二进制字节流
2. 将其（静态存储接哦故）放在运行时数据区的方法区内
3. 内存中（堆内）创建一个这个类的java.lang.Class对象，用来封装类在方法区类的对象

看下图：
  ![此处输入图片的描述][1]
  ![此处输入图片的描述][2]
类的加载的最终产品是位于堆区中的Class对象。

Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。

加载类的方式有以下几种：
 1）从本地系统直接加载
2）通过网络下载.class文件
3）从zip，jar等归档文件中加载.class文件
4）从专有数据库中提取.class文件
5）将Java源文件动态编译为.class文件（服务器）

## 类的初始化

在准备阶段，变量已经赋值过一次初始值。在初始化阶段是执行类构造器<clinit>()方法的过程

只有这6中情况才会导致类的类的初始化：
1. 创建类的实例，也就是new一个对象，或者碰到getstatic、putstatic、invokestatic这几个字节指令码时
2. 访问某个类或接口的静态变量，或者对该静态变量赋值
3. 调用类的静态方法
4. 反射（Class.forName("com.lyj.load")）
5. 初始化一个类的子类，如果父类没有被初始化，会首先初始化子类的父类
6. JVM启动时标明的启动类，含有main方法

**类的初始化步骤：**
1. 如果这个类还没有被加载和链接，那先进行加载和链接
2. 假如这个类存在直接父类，并且这个类还没有被初始化（注意：在一个类加载器中，类只能初始化一次），那就初始化直接的父类（不适用于接口）
3. 加入类中存在静态初始化语句（如static变量和static块），那就依次执行这些初始化语句。

## 类加载器

初始化之前，用户唯一可控的，就是选择类加载器。
JVM的类加载是通过ClassLoader及其子类来完成的，类的层次关系和加载顺序可以由下图来描述：
![此处输入图片的描述][3]
1. Bootstrap ClassLoader
负责加载 `$JAVA_HOME`中jre/lib/rt.jar里所有的class，由C++实现，不是ClassLoader子类
2. Extension ClassLoader
负责加载java平台中扩展功能的一些jar包，包括$JAVA_HOME中jre/lib/*.jar或-Djava.ext.dirs指定目录下的jar包
3. App ClassLoader
负责记载classpath中指定的jar包及目录中class
4. Custom ClassLoader
属于应用程序根据自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现ClassLoader
加载过程中会先检查类是否被已加载，检查顺序是自底向上，从Custom ClassLoader到BootStrap ClassLoader逐层检查，只要某个classloader已加载就视为已加载此类，保证此类只所有ClassLoader加载一次。而加载的顺序是自顶向下，也就是由上层来逐层尝试加载此类。

  [1]: http://o6ubi7qg2.bkt.clouddn.com/classloader.png
  [2]: http://o6ubi7qg2.bkt.clouddn.com/classloader2.png
  [3]: http://o6ubi7qg2.bkt.clouddn.com/classloader3.png