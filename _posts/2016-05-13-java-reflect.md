---
layout: post
title: 深入浅出Java反射
categories: JAVA
description: Java反射, 编译, 加载
keywords: 反射, 编译, 加载
---


# 1. Class类

- 在面向对象的世界里，万事万物皆对象。

Java语言中，基本数据类型（int->Integer包装类）、静态成员（不属于对象，属于类）是对象吗？
类是谁的对象？（Student类是对象，是java.lang.Class类的实例对象）

- 这个对象到底如何表示呢？参看以下code

```
public class ClassDemo1 {
public static void main(String[] args) {
//Foo的实例对象如何表示
Foo foo1 = new Foo();//foo1就表示出来了，编译时加载类（静态加载类）
//Foo这个类，也是一个实例对象，Class类的实例对象，如何表示呢？
//任何一个类都是Class的实例对象，这个实例对象有三种表达方式

//第一种表达方式-->实际告诉我们，任何一个类都有一个隐含的静态成员变量class
Class c1 = Foo.class;

//第二种表达方式，已经知道该类的对象通过getClass方法
Class c2 = foo1.getClass();

/*官网c1,c2表示了Foo类的类类型（class type）
* 万事万物皆对象
* 类也是对象，是Class类的实例对象
* 这个对象也称为该类的类类型
*
*/

//不管c1 or c2都代表了Foo类的类类型，一个类只能是Class类的实例对象
System.out.println(c1 == c2); //true

//第三种表达方式
Class c3 = null;
try {
c3 = Class.forName("com.imooc.reflect.Foo"); //类的全称，运行时加载类（动态加载类）
} catch (ClassNotFoundException e) {
// TODO Auto-generated catch block
e.printStackTrace();
}
System.out.println(c2==c3); //true

//我们完全可以通过类的类类型创建该类的对象实例--->通过c1,c2,c3创建Foo类的实例
try {
Foo foo = (Foo)c1.newInstance();//需要有无参数的构造方法
foo.print();
} catch (InstantiationException e) {
// TODO Auto-generated catch block
e.printStackTrace();
} catch (IllegalAccessException e) {
// TODO Auto-generated catch block
e.printStackTrace();
}
}
}
class Foo{

void print(){
System.out.println("foo");
}
}

public class ClassDemo2 {
public static void main(String[] args) {

Class c1 = int.class;//int 的类类型
Class c2 = String.class;//String类的类类型 String类字节码（作者发明）
Class c3 = double.class;
Class c4 = Double.class;
Class c5 = void.class;

System.out.println(c1.getName()); //int
System.out.println(c2.getName()); //java.lang.String
System.out.println(c2.getSimpleName()); //不包含包名的类的名称
System.out.println(c5.getName()); //void
}

}
```

# 2. 简述编译与加载

- 编译时 会涉及到类加载吗？

  编译的时候、运行的时候加载类是不同的两个概念。
  重点需要关注的时候是运行的时候在什么情况下会触发类的加载
  java需要加载一个class的时候只有在下面集中情况下才会发生：
 1. 创建一个java实例
 2. 调用一个类的静态方法
 3. 静态域赋值
 4. 访问静态域
 5. 在顶层类中执行assert语句

指定查找用户类文件和注释处理程序的位置加载的类是你需要编译类所依赖的类，如你使用了System这个类，因为在jdk里的lib已经存在了，所以你不用显示的如加载，已经在classpath下面了。如果你自己写的一个Class1，把它编译后，再写了个Class2。再编译的时候就需要把Class1的编译文件加载到classpath中。

java中, 编译就是解析源代码生成字节码的过程. 所有的源代码都会被编译成对应的字节码指令. 在这个过程中, 某些语句会对其他类库中的类产生引用, 那么编译器为了保证语句的正确, 必须通过加载该被引用的类来进行验证。由于java的默认单根继承机制(Object), 编译时加载类是无法避免的。

- JAVA 是动态性的解释型编程语言

Java 语言是一种具有动态性的解释型编程语言，当指定程序运行的时候， Java 虚拟机就将编译生成的 . class 文件按照需求和一定的规则加载进内存，并组织成为一个完整的 Java 应用程序。 Java 语言把每个单独的类 Class 和接口 Implements 编译成单独的一个 . class 文件，这些文件对于 Java 运行环境来说就是一个个可以动态加载的单元。

正是因为 Java 的这种特性，我们可以在不重新编译其它代码的情况下，只编译需要修改的单元，并把修改文件编译后的 . class 文件放到 Java 的路径当中， 等到下次该 Java 虚拟机器重新激活时，这个逻辑上的 Java 应用程序就会因为加载了新修改的 .class 文件，自己的功能也做了更新，这就是 Java 的动态性。

下面用一个简单的例子让大家对 Java 的动态加载有一个基本的认识：

```
class TestClassA{
public void method(){
  System.out.println("Loading ClassA");
}
}
public class ClassLoaderTest {
public static void main(String args[]){
  TestClassA testClassA = new TestClassA();
  testClassA.method();
}
}
```

编译后输入命令： java -verbose:class ClassLoaderTest ，执行文件。

JRE （ JavaRuntime Environment ）首先加载 ClassLoaderTest 文件，然后再加载 TestClassA 文件，从而实现了动态加载。

- 预先加载与依需求加载

Java 运行环境为了优化系统，提高程序的执行速度，在 JRE 运行的开始会将 Java 运行所需要的基本类采用预先加载（ pre-loading ）的方法全部加载要内存当中，因为这些单元在 Java 程序运行的过程当中经常要使用的，主要包括 JRE 的 rt.jar 文件里面所有的 .class 文件。

 当 java.exe 虚拟机开始运行以后，它会找到安装在机器上的 JRE 环境，然后把控制权交给 JRE ， JRE 的类加载器会将 lib 目录下的 rt.jar 基础类别文件库加载进内存，这些文件是 Java 程序执行所必须的，所以系统在开始就将这些文件加载，避免以后的多次 IO 操作，从而提高程序执行效率。我们可以看到多个基础类被加载， java.lang.Object,java.io.Serializable 等等。

相对于预先加载，我们在程序中需要使用自己定义的类的时候就要使用依需求加载方法（ load-on-demand ），就是在 Java 程序需要用到的时候再加载，以减少内存的消耗，因为 Java 语言的设计初衷就是面向嵌入式领域的。

在这里还有一点需要说明的是， JRE 的依需求加载究竟是在什么时候把类加载进入内部的呢？

我们在定义一个类实例的时候，比如 TestClassA testClassA ，这个时候 testClassA 的值为 null ，也就是说还没有初始化，没有调用 TestClassA 的构造函数，只有当执行 testClassA = new TestClassA() 以后， JRE 才正真把 TestClassA （运行时）加载进来。

- 隐式加载和显示加载

Java 的加载方式分为隐式加载（ implicit ）和显示加载（ explicit ），上面的例子中就是用的隐式加载的方式。所谓隐式加载就是我们在程序中用 new 关键字来定义一个实例变量， JRE 在执行到 new 关键字的时候就会把对应的实例类加载进入内存。隐式加载的方法很常见，用的也很多， JRE 系统在后台自动的帮助用户加载，减少了用户的工作量，也增加了系统的安全性和程序的可读性。无法进行动态加载，new的这个 Class 对象必须是你程序代码编译的时候有的。

显示加载可以实现动态加载，你可以在程序运行时加载进一个编译时并没有的类。就是有程序员自己写程序把需要的类加载到内存当中，下面我们看一段程序：

```
class TestClass{
public void method(){
  System.out.println("TestClass-method");
}
}

public class CLTest {
public static void main(String args[]) {
  try{
  Class c = Class.forName("TestClass");
  TestClass object = (TestClass)c.newInstance();
  object.method();
  }catch(Exception e){
  e.printStackTrace();
  }
}
}
```

Class.forName(xxx.xx.xx)返回的是一个类Class.forName(xxx.xx.xx)的作用是要求JVM查找并加载指定的类，也就是说JVM会执行该类的静态代码段。在初始化一个类，生成一个实例的时候，newInstance()方法和new关键字除了一个是方法，一个是关键字外，最主要有什么区别？

它们的区别在于创建对象的方式不一样，前者是使用类加载机制，后者是创建一个新类。那么为什么会有两种创建对象方式？这主要考虑到软件的可伸缩、可扩展和可重用等软件设计思想。

Java中工厂模式经常使用newInstance()方法来创建对象，因此从为什么要使用工厂模式上可以找到具体答案。 例如：
```
class c = Class.forName(“Example”);
factory = (ExampleInterface)c.newInstance();
```
其中ExampleInterface是Example的接口，可以写成如下形式：
```
String className = “Example”;
class c = Class.forName(className);
factory = (ExampleInterface)c.newInstance();
```
进一步可以写成如下形式：
```
String className = readfromXMlConfig;
//从xml 配置文件中获得字符串
class c = Class.forName(className);
factory = (ExampleInterface)c.newInstance();
```

上面代码已经不存在Example的类名称，它的优点是，无论Example类怎么变化，上述代码不变，甚至可以更换Example的兄弟类Example2 , Example3 , Example4……，只要他们继承ExampleInterface就可以。

从JVM的角度看，我们使用关键字new创建一个类的时候，这个类可以没有被加载。但是使用newInstance()方法的时候，就必须保证：1、这个类已经加载；2、这个类已经连接了。 而完成上面两个步骤的正是Class的静态方法forName()所完成的，这个静态方法调用了启动类加载器，即加载 java API的那个加载器。

现在可以看出，newInstance()实际上是把new这个方式分解为两步，即首先调用Class加载方法加载某个类，然后实例化。 这样分步的好处是显而易见的。我们可以在调用class的静态加载方法forName时获得更好的灵活性，提供给了一种降耦的手段。

- new 和Class.forName( )什么区别？

首先，newInstance( )是一个方法，而new是一个关键字； 其次，Class下的newInstance()的使用有局限，因为它生成对象只能调用无参的构造函数，而使用 new关键字生成对象没有这个限制。 简言之：newInstance(): 弱类型,低效率,只能调用无参构造。new: 强类型,相对高效,能调用任何public构造。Class.forName( )返回的是类。Class.forName( ).newInstance()返回的是object。

# 3. 获取类类型后，基本API（方法、成员变量、构造函数）

- Class类的基本API

```
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class ClassUtil {
/**
* 打印类的信息，包括类的成员函数
* @param obj 该对象所属类的信息
*/
public static void printClassMethodMessage(Object obj){
//要获取类的信息，首先获取其类类型
Class c = obj.getClass();//传递的是哪个子类的对象 c就是该子类的类类型，通过Native方法实现
//获取类的名称
System.out.println("类的名称是:"+c.getName());
/*
* Method类，方法对象
* 一个成员方法就是一个Method对象
* getMethods()获取所有public函数，包括从父类继承来的
* getDeclaredMethods()获取所有权限函数，只有自己声明的方法
*/
Method[] ms = c.getMethods();//c.getDeclaredMethods()
for(int i = 0; i < ms.length;i++){
//得到方法的返回值类型的类类型
Class returnType = ms[i].getReturnType();
System.out.print(returnType.getName()+" ");
//得到方法名
System.out.print(ms[i].getName()+"(");
//获取参数类型-->得到的是参数列表类型的类类型
Class[] paramTypes = ms[i].getParameterTypes();
for (Class class1 : paramTypes) {
System.out.print(class1.getName()+",");
}
System.out.println(")");
}
}
  /**
  * 获取成员变量信息
  * @param obj
  */
public static void printFieldMessage(Object obj) {
Class c = obj.getClass();
/*
* 成员变量也是对象
* java.lang.reflect.Field
* Field类封装了关于成员变量的操作
* getFields()方法获取所有public的成员变量的信息
* getDeclaredFields获取该类自己声明的成员变量信息
*/
//Field[] fs = c.getFields();
Field[] fs = c.getDeclaredFields();
for (Field field : fs) {
//得到成员变量的类型的类类型
Class fieldType = field.getType(); // String.class
String typeName = fieldType.getName();
//得到成员变量的名称
String fieldName = field.getName();
System.out.println(typeName+" "+fieldName);
}
}
/**
* 打印构造函数信息
* @param obj
*/
public static void printConMessage(Object obj){
Class c = obj.getClass(); //先获取类类型
/*
* 构造函数也是对象
* java.lang. Constructor类封装了关于构造函数的操作
* getConstructors获取所有public的构造方法
* getDeclaredConstructors获取自己声明的构造函数
*/
//Constructor[] cs = c.getConstructors();
Constructor[] cs = c.getDeclaredConstructors();
for (Constructor constructor : cs) {
System.out.print(constructor.getName()+"(");
//获取构造函数的参数列表-->得到的是参数列表的类类型
Class[] paramTypes = constructor.getParameterTypes();
for (Class class1 : paramTypes) {
System.out.print(class1.getName()+",");
}
System.out.println(")");
}
}
}

```

# 4. 方法的反射

- 如何获取某个方法
  方法的名称和参数列表才能唯一决定某个方法
- 方法的反射操作
  method.invoke(对象，参数列表)

```
import java.lang.reflect.Method;

public class MethodDemo1 {
public static void main(String[] args) {
//要获取print(int,int)方法
//1. 要获取一个方法就是获取类的信息，首先要获取类类型
A a1 = new A();
Class c = a1.getClass();
/*
* 2. 获取方法 名称和参数列表来决定
* getMethod获取的是所有的public方法
* getDelcaredMethod获取自己声明的方法
*/
try {
//Method m = c.getMethod("print", new Class[]{int.class,int.class});
Method m = c.getMethod("print", int.class,int.class);

//方法的反射
//a1.print(10, 20);方法的反射使用m对象来进行方法调用 和a1.print调用效果一样
//方法如果没有返回值，返回null，有返回值则返回具体的返回值
//Object o = m.invoke(a1,new Object[]{10,20});
Object o = m.invoke(a1, 10,20);
System.out.println("==================");
//获取方法print(String,String)
  Method m1 = c.getMethod("print",String.class,String.class);
  //用方法进行反射操作
  //a1.print("hello", "WORLD");
  o = m1.invoke(a1, "hello","WORLD");
  System.out.println("===================");
  // Method m2 = c.getMethod("print", new Class[]{});
  Method m2 = c.getMethod("print");
  // m2.invoke(a1, new Object[]{});
  m2.invoke(a1);
} catch (Exception e) {
// TODO Auto-generated catch block
e.printStackTrace();
}

}
}
class A{
public void print(){
System.out.println("helloworld");
}
public void print(int a,int b){
System.out.println(a+b);
}
public void print(String a,String b){
System.out.println(a.toUpperCase()+","+b.toLowerCase());
}
}
```

# 5.通过反射了解集合泛型的本质
通过Class，Method来认识泛型的本质

```

import java.lang.reflect.Method;
import java.util.ArrayList;

public class MethodDemo4 {
public static void main(String[] args) {
ArrayList list = new ArrayList();

ArrayList<String> list1 = new ArrayList<String>();
list1.add("hello");
//list1.add(20);格式不对
Class c1 = list.getClass();
Class c2 = list1.getClass();
System.out.println(c1 == c2); //true
//反射的操作都是编译之后的操作

/*
* c1==c2结果返回true说明编译之后集合的泛型是去泛型化的
* Java中集合的泛型，是防止错误输入，只在编译阶段有效，
* 绕过编译检测就无效了
* 验证：我们可以通过方法的反射来操作，绕过编译
*/
try {
Method m = c2.getMethod("add", Object.class);
m.invoke(list1, 20);//绕过编译检测就绕过了泛型，加进去了！
System.out.println(list1.size());
System.out.println(list1);
/*for (String string : list1) {
System.out.println(string);
}*///现在不能这样遍历了，里面保存多种数据类型
} catch (Exception e) {
e.printStackTrace();
}
}

}
```