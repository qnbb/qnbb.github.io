---
layout: post
title: Spring Bean的作用域
categories: Spring
description: 5种作用域
keywords: Singleton, Prototype, request, session, global session 
---

### 5种作用域

|作用域 |描述|
|:--:|:--|
|singleton|在每个Spring IoC容器中一个bean定义对应一个对象实例。|
|prototype|一个bean定义对应多个对象实例。|
|request|在一次HTTP请求中，一个bean定义对应一个实例；即每次HTTP请求将会有各自的bean实例， 它们依据某个bean定义创建而成。该作用域仅在基于web的Spring ApplicationContext 情形下有效。
|session|在一个HTTP Session 中，一个bean定义对应一个实例。该作用域仅在基于web的SpringApplicationContext 情形下有效。|
|global session|在一个全局的HTTP Session 中，一个bean定义对应一个实例。典型情况下，仅在使用portlet context的时候有效。该作用域仅在基于web的Spring ApplicationContext 情形下有效。|

有三种只能用在基于web的Spring ApplicationContext：

### Singleton作用域

当一个bean的作用域为singleton, 那么Spring IoC容器中只会存在一个共享的bean实例，并且所有对bean的请求，只要id与该bean定义相匹配，则只会返回bean的同一实例。换言之，当把一个bean定义设置为singlton作用域时，Spring IoC容器只会创建该bean定义的唯一实例。这个单一实例会被存储到单例缓存（singleton cache）中，并且所有针对该bean的后续请求和引用都将返回被缓存的对象实例。

![此处输入图片的描述][1]

把Spring的singleton作用域描述成一个Container (容器)对应一个bean 实例最为贴切。也就是说假如在单个Spring容器内定义了某个指定class的bean，那么Spring容器将会创建一个并且仅有一个 由该bean定义指定的类实例。Singleton作用域是Spring中的`缺省作用域` 。要在XML中将bean定义成singleton，可以这样配置：
```
<bean id="accountService" class="com.foo.DefaultAccountService"/>

<!-- the following is equivalent, though redundant (singleton scope is the default); using spring-beans-2.0.dtd or upper-->
<bean id="accountService" class="com.foo.DefaultAccountService" scope="singleton"/>

<!-- the following is equivalent and preserved for backward compatibility in spring-beans.dtd -->
<bean id="accountService" class="com.foo.DefaultAccountService" singleton="true"/>
```

 beans.xml配置文件中配置如下所示：
```
<bean id="personService" class="examples.test.PersonServiceBean" ></bean>
```
测试代码如下所示：
```
public class SpringTest {

@Test
public void instanceSpring(){
  ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
  PersonService personService1 = (PersonService)ctx.getBean("personService");
  PersonService personService2 = (PersonService)ctx.getBean("personService");
  System.out.println(personService1==personService2);
  }
}
```
 运行测试程序，结果输出为:true，说明是只创建了一个PersonServiceBean。

### Prototype作用域

Prototype作用域的bean会导致在每次对该bean请求（将其注入到另一个bean中，或者以程序的方式调用容器的getBean() 方法）时都会创建一个新的bean实例 。根据经验，对有状态的bean应该使用prototype作用域，而对无状态的bean则应该使用singleton作用域。

下图演示了Spring的prototype作用域。请注意，通常情况下，DAO不会被配置成prototype，因为DAO通常不会持有任何会话状态，因此应该使用singleton作用域。

![此处输入图片的描述][2]

在XML中将bean定义成prototype，可以这样配置：
```
<!-- using spring-beans-2.0.dtd or upper -->
<bean id="accountService" class="com.foo.DefaultAccountService" scope="prototype"/>
<!-- the following is equivalent and preserved for backward compatibility in spring-beans.dtd -->
<bean id="accountService" class="com.foo.DefaultAccountService" singleton="false"/>
```
 将beans.xml配置文件中的配置信息改为如下所示：
```
<bean id="personService" class="cn.itcast.service.impl.PersonServiceBean" scope="prototype"></bean>
```
再运行上面的测试程序，输出结果为:false，说明创建了两个PersonServiceBean。

### 其他作用域

request、session以及global session 仅在基于web的应用中使用（不必关心你所采用的是什么web应用框架）。这些作用域仅仅在使用基于web的Spring ApplicationContext实现（如XmlWebApplicationContext）时有用。 如果在普通的Spring IoC容器中，比如像XmlBeanFactory或ClassPathXmlApplicationContext， 尝试使用这些作用域，你将会得到一个IllegalStateException异常（未知的bean作用域）。

引用：[Bean的作用域及生命周期][3]、[Spring 实战][4]

  [1]: http://o6ubi7qg2.bkt.clouddn.com/springsingleton.png
  [2]: http://o6ubi7qg2.bkt.clouddn.com/springprototype.png
  [3]: http://blog.csdn.net/feihong247/article/details/7798474
  [4]: https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B00CY6UD2I?ie=UTF8&ref_=cm_cr_arp_d_product_top