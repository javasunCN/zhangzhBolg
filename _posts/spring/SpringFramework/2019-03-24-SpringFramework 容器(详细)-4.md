---
layout: post
title:  "Spring源码阅读[四]"
date:   2019-3-24 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[四]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>Spring容器</h3></b></center>

## **核心类介绍** 
      
### **DefaultListableBeanFacto1y** 
{% highlight text %}
**介绍**：
    DefaultListableBeanFactory 是整个bean加载的**核心**部分,Spring注册及加载 Bean 的默认实现。如图1
    (
        XmlBeanFactory继承DefaultListableBeanFactory
        不同：
            *   XmlBeanFactory 使用自定义的 XML 读取器 XmlBeanDefinitionReader，
                实现个性化BeanDefinitionReader读取
            *   DefaultListableBeanFactory 继承 AbstractAutowireCapableBeanFactory,
                实现ConfigurableListableBeanFactory、BeanDefinitionRegistry
    )
    
**容器加载类说明**： 
    * AliasRegistry(接口): Bean别名增删查操作
    * SimpleAliasRegistry(实现类): 简单实现AliasRegistry接口方法定义，使用线程安全的Map实现 **别名** 缓存;
    * BeanDefinitionRegistry(接口): 对BeanDefinition的增删改查的操作方法
    * SingletonBeanRegistry: 定义单例Bean操作方式
    * BeanFactory: 定义操作Bean的方法，获取Bean的实例、别名以及属性
    * DefaultSingletonBeanRegistry: 继承SimpleAliasRegistry,实现SingletonBeanRegistry,对SingletonBean的具体操作
    * ListableBeanFactory: 继承BeanFactory,在BeanFactory中枚举快速查找Bean操作定义
    * HierarchicalBeanFactory: 继承BeanFactory, 分层实现BeanFactory,获取工厂的父级工厂
    
    * ConfigurableBeanFactory: [BeanFactory配置核心类]配置BeanFactory, 继承分层BeanFactory和单例Bean注册
    * FactoryBeanRegistrySupport: 扩展DefaultSingletonBeanRegistry功能实现对FactoryBean的特殊处理

    * AutowireCapableBeanFactory: 继承BeanFactory, 创建Bean、自动注入、初始化以及Bean的后置处理器
    * AbstractBeanFactory: 继承FactoryBeanRegistrySupport，实现ConfigurableBeanFactory，
        操作BeanFactory扩展，实现FactoryBeanRegistrySupport、ConfigurableBeanFactory所有功能
    * ConfigurableListableBeanFactory:Beanfactory配置清单，指定忽略类型及接口等
    * AbstractAutowireCapableBeanFactory:继承 AbstractBeanFacto1y 并对接口 AutowireCapableBeanFactory进行实现
        加载配置文件、实例化Bean
    * DefaultListableBeanFactory: 综合以上的所有功能，对Bean的后置处理


{% endhighlight %} 
![图1(容器加载类关系图)]({{ site.url }}{{site.baseurl}}/assets/img/spring/WechatIMG271.png)

> 注:XmlBeanFactory对 DefaultListableBeanFactorγ类进行了扩展，
> 主要用于从 XML 文档中读取 BeanDefinition，对于注册及获取 bean 都是使用从父类 DefaultListableBeanFactory 继承的方法去实现，
> 而唯独与父类不同的个性化实现就是增加了 XmlBeanDefinitionReader类型的 reader 属性。
> 在 XmlBeanFactory 中主要使用 reader属性对资源文件进行读取和注册。

### **XmlBeanDefinitionReader** 
{% highlight text %}
**介绍**：
    XmlBeanDefinitionReader主要负责XML文件的读取、解析以及注册
 
**容器加载类说明**：
    * ResourceLoader: 定义资源加载器，解析给定的资源文件位置返回Resource对象
    * BeanDefinitionReader: 读取给定的资源文件并转换为BeanDefinition的各种功能
    * EnvironmentCapable: 定义获取Environment参数的方法
    * DocumentLoader: 定义从资源文件中读取转换为Document的功能
    * AbstractBeanDefinitionReader: 对BeanDefinitionReader、EnvironmentCapable定义的功能实现
    * BeanDefinitionDocumentReader: 定义读取 Docuemnt 并注册 BeanDefinition 功能
        SPI(service provider interface JDK内置的中服务发现机制)用于解析包含Spring bean定义的XML文档。
    * BeanDefinitionParserDelegate:定义解析 Element 的各种方法 。
    
{% endhighlight %} 

![配置访问相关类]({{ site.url }}{{site.baseurl}}/assets/img/spring/配置访问相关类.png)

{% highlight text %}
处理步骤:
    1、通过继承自 AbstractBeanDefinitionReader中的方法，来使用ResourLoader将资源文件路径转换为对应的Resource文件。
    2、通过 DocumentLoader 对 Resource 文件进行转换，将 Resource 文件转换为 Document 文件。
    3、通过实现接口 BeanDefinitionDocumentReader 的 DefaultBeanDefinitionDocumentReader类
       对 Document进行解析，并使用 BeanDefinitionParserDelegate对 Element进行解析。
{% endhighlight %}


