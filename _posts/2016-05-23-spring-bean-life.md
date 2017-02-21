---
layout: post
title: Spring Bean的生命周期
categories: Spring
description: 什么时候初始化bean实例,Bean 从创建到销毁
keywords: BeanFactory
---

大家应该知道spring 中有几种供 springbean 生存的容器：

- Bean工程
  - BeanFactory
- 应用上下文(ApplicationContext)
  - ClassPathXmlApplicationContext
  - FilesSystemXMLApplicationContext
  - webApplicationContext

这里就只介绍BeanFactory 和 ApplicationContext 。另外ApplicationContext其实是基于BeanFactory上发展来的。

理解springBean 的生命周期主要通过两个层面来理解。其一是 [Bean 的作用域][1]，其二是实例化 Bean 时所经历的一系列阶段。

## 什么时候初始化bean实例
- lazy-init（默认false）

我们可以指定Bean节点的 lazy-init="true" 来延迟初始化bean，这时候，只有第一次获取bean才会初始化bean，即第一次请求该bean时才初始化。如下配置所示：
```
<bean id=”xxx” class=”examples.test.OrderServiceBean” lazy-init=”true” />
```
如果想对所有bean都应用延迟初始化，可以在根节点beans设置default-lazy-init=”true”,如下所示：
```
<beans default-lazy-init=”true” …>
```
- scope的影响

scope = singleton ,即默认情况，会在容器初始化时实例化。

scope = prototype ,会延迟初始化bean,即第一次请求该bean时才初始化(如调用getBean()方法时)。

测试如下：PersonServiceBean代码如下所示：
```
public class PersonServiceBean implements PersonService {
  public void init(){
  System.out.println("Get the database connection and initialize other beans!");
  }
  public PersonServiceBean(){
  System.out.println("PersonServiceBean is initialized!");
  }
  public void destroy(){
  System.out.println("close the database connection!");
  }
}
```
 beans.xml配置文件如下所示（作用域为singleton）：
```
<!-- 1 -->
<bean id="personService" class="examples.test.PersonServiceBean" /> <!-- 2 -->
<bean id="personService" class="examples.test.PersonServiceBean" scope="singleton" />
```
测试程序如下所示：
```
public class SpringTest {
@Test
 public void instanceSpring(){
  ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
  System.out.println("------------------");
  PersonService personService = (PersonServiceBean)ctx.getBean("personService");
}
}
```
 分1，2两种情况运行程序，控制台打印信息如下所示：
```
PersonServiceBean is initialized!
------------------
```
将配置文件修改成如下所示（设置懒加载、作用域为prototype）：
```
<!-- 1 -->
<bean id="personService" class="examples.test.PersonServiceBean" lazy-init="true" />
<!-- 2 -->
<bean id="personService" class="examples.test.PersonServiceBean" scope="prototype" />
```
分1，2两种情况运行程序，控制台打印信息如下所示：
```
------------------
PersonServiceBean is initialized!
```
以上测试说明：

Bean默认是在容器初始化时初始化的（lazy-init = false），即ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");语句执行时就实例化bean了。如果把scope设成scope=”prototype”或设置lazy-init=”true”，则会延迟bean的实例化，bean会在PersonService personService = (PersonServiceBean)ctx.getBean("personService");语句执行时才实例化。

当配置文件为如下所示时，测试结果同上。
```
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:context= "http://www.springframework.org/schema/context"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
  http://www.springframework.org/schema/context
  http://www.springframework.org/schema/context/spring-context-2.5.xsd"
  default-lazy-init="true"
>
  <bean id="personService" class="cn.itcast.service.impl.PersonServiceBean" >
</beans>
```

- scope = prototype 与 destory()

如果bean的scope设为prototype时，当ctx.close时，destroy方法不会被调用。原因如下：

1. 对于prototype作用域的bean，有一点非常重要，那就是Spring不能对一个prototype bean的整个生命周期负责：容器在初始化、配置、装饰或者是装配完一个prototype实例后，将它交给客户端，随后就对该prototype实例不闻不问了。
2. 不管何种作用域，容器都会调用所有对象的初始化生命周期回调方法。但对prototype而言，任何配置好的析构生命周期回调方法都将不会被调用。清除prototype作用域的对象并释放任何prototype bean所持有的昂贵资源，都是客户端代码的职责。（让Spring容器释放被prototype作用域bean占用资源的一种可行方式是，通过使用bean的后置处理器，该处理器持有要被清除的bean的引用）
3. 谈及prototype作用域的bean时，在某些方面你可以将Spring容器的角色看作是Java new 操作的替代者。任何迟于该时间点的生命周期事宜都得交由客户端来处理。

配置文件信息如下：
```
<bean id="personService" class="cn.itcast.service.impl.PersonServiceBean" init-method="init" destroy-method="destroy"/>
```
  测试程序如下：
```
public class SpringTest {
@Test
public void instanceSpring(){
  AbstractApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
  System.out.println("------------------");
  PersonService personService = (PersonServiceBean)ctx.getBean("personService");
  ctx.close();
}
```
运行测试程序，控制台打印信息如下所示：
```
PersonServiceBean is initialized!
Get the database connection and initialize other beans!
------------------
close the database connection!
```
修改配置文件信息如下：
```
<bean id="personService" class="cn.itcast.service.impl.PersonServiceBean" scope="prototype" init-method="init" destroy-method="destroy"/>
```
再次运行测试程序，控制台打印信息如下所示：
```
------------------
PersonServiceBean is initialized!
Get the database connection and initialize other beans!
```
修改配置文件信息如下所示：
```
<bean id="personService" class="cn.itcast.service.impl.PersonServiceBean" lazy-init="true" init-method="init" destroy-method="destroy"/>
```
此时再次运行测试程序，控制台打印信息如下所示：
```
------------------
PersonServiceBean is initialized!
Get the database connection and initialize other beans!
close the database connection!
```
## Bean 从创建到销毁
下图描述了`BeanFactory` 中 bean 生命周期的完整过程

![此处输入图片的描述][2]

1. 当调用者通过 getBean（ name ）向 容器寻找Bean 时，如果容器注册了org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor接口，在实例 bean 之前，将调用该接口的 postProcessBeforeInstantiation （）方法。

2. 根据配置情况调用 Bean构造函数或工厂方法实例化 bean

3. 如果容器注册了 org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor接口，在实例 bean 之后，调用该接口的 postProcessAfterInstantiation （）方法，可以在这里对已经实例化的对象进行一些装饰。

4. 受用依赖注入，Spring 按照 Bean 定义信息配置 Bean 的所有属性，在设置每个属性之前将调用InstantiationAwareBeanPostProcess接口的 postProcessPropertyValues （）方法 。

5. 如果 Bean 实现了 BeanNameAware 接口，工厂调用 Bean 的 setBeanName() 方法传递 Bean 的 ID 。

6. 如果 Bean 实现了 BeanFactoryAware 接口，工厂调用setBeanFactory()方法传入工厂自身。

7. 如果 BeanPostProcessor和Bean关联，那么将调用该接口的 postProcessBeforeInitialzation() 方法 对 bean进行加工操作，这个非常重要，spring 的 AOP 就是用它实现的。

8. 如果bean 实现了 InitializingBean 接口，将调用 afterPropertiesSet() 方法

9. 如果Bean 指定了 init-method 方法，它将被调用。

10. 如果有BeanPsotProcessor 和 Bean 关联，那么它们的 postProcessAfterInitialization() 方法将被调用。 到这个时候， Bean 已经可以被应用系统使用了 。

11. 如果在<bean> 中指定了该 bean 的作用范围为 scope="prototype", 将 bean 的调用者，调用者管理该 bean 的生命周期， spring 不在管理该 bean 。

12. 如果在<bean> 中指定了该 bean 的作用范围为 scope="singleton", 则将该 bean 放入 springIOC 的缓存池中，将触发 spring 对该 bean 的生命周期管理。

13. 当容器关闭时，Bean 从Bean Factory销毁，还可能 ：
  - 如果 Bean 实现了 DisposableBean 接口， destory() 方法被调用。

  - 如果指定了订制的销毁方法,使用destroy-method声明了销毁方法，就调用这个方法。

另外`ApplicationContext`和BeanFactory 只是多了一个接口。如果Bean 实现了 ApplicationContextAwre 接口， setApplicationContext() 方法被调用。

## ？
1. 如果不调用任何init,destroy之类的方法，对容器来说，只是少一些个性定制化方法吗？
  答：是的（我猜）
2. 在scope = singleton，在其销毁Bean时，才触发了 destroy() 之类的方法。destroy方法并不会加速其销毁速度？
  答：是的（我猜）

----------------------

引用：[Bean的作用域及生命周期][3]、[Spring 实战][4]、[Spring Bean 生命周期][5]

  [1]: http://monkeyhorse.cn/posts/16
  [2]: http://o6ubi7qg2.bkt.clouddn.com/springbeanscope.jpg
  [3]: http://blog.csdn.net/feihong247/article/details/7798474
  [4]: https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B00CY6UD2I?ie=UTF8&ref_=cm_cr_arp_d_product_top
  [5]: http://997004049-qq-com.iteye.com/blog/1729793