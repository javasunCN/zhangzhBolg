---
layout: post
title:  "Spring源码阅读[二]"
date:   2019-3-21 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[二]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>模块介绍</h3></b></center>

#### **Core Container (核心容器) 模块**

          
{% highlight text %}
1、Core Container (核心容器)包含：Core、 Beans、 Context和 ExpressionLanguage模块
    1.1、Core 和 Beans 模块是框架的基础，提供 IoC (转控制)和依赖注入特性（基础：BeanFactory）
        1.1.1、Core 模决主要包含 Spring 框架基本的核心工具类，Core 模块是其他纽件的基本核心 。
        1.1.2、Beans 模块是所有应用都妥用到的，它包含访问配直文件、创建和管理 bean 以及进行 
                Inversion of Control I Dependency Injection ( IoC/DI )操作相关的所有类 。
        1.1.3、Context模块构建于 Core 和 Beans模块基础之上，提供了一种类似于JNDI注册器的框架式的对象访问方法。
                Context 模块继承了 Beans 的特性，为 Spring 核心提供了大量 扩展，
                添加了对国际化(例如资源绑定)、事件传播、资源加载和对 Context 的 透明创 建的支持。 
                ApplicationContext 接口是 Context 模块的关键.
        1.1.4、Expression Language模块提供了强大的表达式语言，用于在运行时查询和操纵对象。
                该语言支持设直/获取属 性的值，属性的分配，方法的调用，访问数组上下文( accessiong the context of arrays )、 
                容器和索引器、逻辑和算术运算符、命名变量以及从 Sprit屯的 IoC 容器中根据名称检 索对象 。 
                它也支持 list 投影、选择和一般的 list 聚合 。
{% endhighlight %} 
    
#### **Data Access/Integration 模块**
{% highlight text %}
1、Data Access/Integration包含：JDBC、 ORM、 OXM、JMS和Transaction模块
    1.1、JDBC 模块提供了一个 JDBC 抽象层，它消除冗长的 JDBC 编码和解析数据库厂
        商特有的错误代码 。 这个模块包含了 Spring 对 JDBC 数据访问进行封装的所有类
    1.2、ORM 模块为流行的对象-关系映射 API，如 JPA、JDO、Hibernate、 iBatis 等，提供了 一个交互层。
    1.3、OXM 模块提供了一个对 ObjectXML 映射实现的抽象层， Object/XML 映射实现包括
        JAXB、 Castor、 XMLBeans、 JiBX 和 XStrearm。
    1.4、JMS ( Java Messaging Service )模块主要包含了一些制造和消费消息的特性 。
    1.5、Transaction 模块支持编程和声明性的事务 管理，这些事务类必须实现特定的接口，
        并且对所有的 POJO 都适用.
    
{% endhighlight %} 

---------------------------------------

#### **WEB 模块**

{% highlight text %}
1、Web模块包含：WebSocket、Servlet、Web、Portlet
    1.1、WebSocket模块：websocket是Html5新增加特性之一，目的是浏览器与服务端建立全双工的通信方式，
        解决http请求-响应带来过多的资源消耗，同时对特殊场景应用提供了全新的实现方式，
        比如聊天、股票交易、游戏等对对实时性要求较高的行业领域。
    1.2、Web模块：提供了基础的面向 Web 的集成特性 
        例如，多文件 上传 、使用 servlet listeners 初始化 IoC 容器以及一个面向 Web 的应用上下文 。 
    1.3、Portlet模块：提供了用于 Portlet环境和 Web-Servlet模块的 MVC 的实现
    1.4、Servlet模块：该模块包含 Spring 的 model-view-controller ( MVC)实现。
        Spring 的 MVC 框架使得模型范围内的代码和 web forms 之间能够清楚地分离开来，
        并与Spring框架的其他特性集成在一起。
    
{% endhighlight %} 


#### **AOP 模块**
{% highlight text %}
1、AOP模块提供了一个符合 AOP联盟标准的面向切面编程的实现，
    它让你可以定义例如方 法拦截器和切点，从而将逻辑代码分开，降低它们之间的调合性 。 
    
{% endhighlight %} 

#### **Aspects 模块**
{% highlight text %}
1、Aspects 模块:提供切面编程(AspectJ)的支持
    
{% endhighlight %} 

#### **Instrumentation 模块**
{% highlight text %}
1、Instrumentation 模块:基于JAVA SE中的"java.lang.instrument"进行设计的，
    应该算是Aop的一个支援模块，主要作用是在JVM启用时，
    生成一个代理类，程序员通过代理类在运行时修改类的字节，
    从而改变一个类的功能，实现Aop的功能。
    
{% endhighlight %} 

#### **Instrumentation 模块**
{% highlight text %}
1、messaging 模块:对基础消息报文的处理支持
    
{% endhighlight %} 

#### **Test 模块**
{% highlight text %}
1、TEST 模块:支持使用 JUnit和 TestNG 对 Spring组件进行测试
    
{% endhighlight %} 

