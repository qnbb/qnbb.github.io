---
layout: post
title: SpringMVC、Struts2主流程
categories: SpringMVC
description: SpringMVC、Struts2主流程
keywords: SpringMVC, Struts2
---

## SpringMVC
### SpringMVC工作流程图
p1：
![此处输入图片的描述][1]

p2：
![此处输入图片的描述][2]

### SpringMVC的工作流程描述:

1. 用户向服务器发送请求，请求被Spring前端控制Servelt DispatcherServlet捕获；
2. DispatcherServlet对请求URL进行解析，得到请求资源标识符（URI）。然后根据该URI，调用HandlerMapping获得该Handler配置的所有相关的对象（包括Handler对象以及Handler对象对应的拦截器），最后以HandlerExecutionChain对象的形式返回；
3. DispatcherServlet根据获得的Handler，选择一个合适的HandlerAdapter。（附注：如果成功获得HandlerAdapter后，此时将开始执行拦截器的preHandler(...)方法）
4. 提取Request中的模型数据，填充Handler入参，开始执行Handler（Controller)。在填充Handler的入参过程中，根据你的配置，Spring将帮你做一些额外的工作：

  HttpMessageConveter：将请求消息（如Json、xml等数据）转换成一个对象，将对象转换为指定的响应信息
  数据转换：对请求消息进行数据转换。如String转换成Integer、Double等
  数据根式化：对请求消息进行数据格式化。如将字符串转换成格式化数字或格式化日期等
  数据验证：验证数据的有效性（长度、格式等），验证结果存储到BindingResult或Error中
5. Handler执行完成后，向DispatcherServlet 返回一个ModelAndView对象；
6. 根据返回的ModelAndView，选择一个适合的ViewResolver（必须是已经注册到Spring容器中的ViewResolver)返回给DispatcherServlet ；
7. ViewResolver结合Model和View，来渲染视图
8. 将渲染结果返回给客户端。

## Struts
### Struts工作流程图
![此处输入图片的描述][3]

### Struts的工作流程描述:

1. 客户端初始化一个指向Servlet容器（例如Tomcat）的请求；

2. 这个请求经过一系列的过滤器（Filter）（这些过滤器中有一个叫做ActionContextCleanUp的可选过滤器，这个过滤器对于Struts2和其他框架的集成很有帮助，例如：SiteMeshPlugin）；

3. 接着StrutsPrepareAndExecuteFilter被调用，StrutsPrepareAndExecuteFilter询问ActionMapper来决定这个请求是否需要调用某个Action；

4. 如果ActionMapper决定需要调用某个Action，StrutsPrepareAndExecuteFilter把请求的处理交给ActionProxy；

5. ActionProxy通过ConfigurationManager询问框架的配置文件，找到需要调用的Action类；

6. ActionProxy创建一个ActionInvocation的实例。

7. ActionInvocation实例使用命名模式来调用，在调用Action的过程前后，涉及到相关拦截器（Intercepter）的调用。

8. 一旦Action执行完毕，ActionInvocation负责根据struts.xml中的配置找到对应的返回结果。返回结果通常是（但不总是，也可能是另外的一个Action链）一个需要被表示的JSP或者FreeMarker的模版。在表示的过程中可以使用Struts2框架中继承的标签。在这个过程中需要涉及到ActionMapper。

  [1]: http://o6ubi7qg2.bkt.clouddn.com/SpringMVC%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B%E5%9B%BE1.png
  [2]: http://o6ubi7qg2.bkt.clouddn.com/SpringMVC%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B%E5%9B%BE2.png
  [3]: http://o6ubi7qg2.bkt.clouddn.com/SpringMVC%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B%E5%9B%BE2.png