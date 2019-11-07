---
layout: post
title:  "Spring源码阅读[三]"
date:   2019-3-23 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[三]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>环境搭建(IntelliJ IDEA详细)</h3></b></center>

#### **源码链接获取**       
{% highlight text %}
1、打开www.github.com,搜索spring，定位：https://github.com/spring-projects/spring-framework
2、打开Idea VCS -> Git -> https://github.com/spring-projects/spring-framework.git (注意：git配置：我的路径为/use/bin/git)
3、Clone spring-framework到本地
4、执行：
    
4、Gradle导入
{% endhighlight %} 

#### **错误问题解决**  
##### **cglib 和 obienesis 的编译错误解决**
{% highlight text %}
1、问题描述：
    为了避免第三方 class的冲突， Spring把最新的 cglib和 obj巳nesis给重新打包( repack)了， 
    它并没有在源码里提供这部分的代码，而是直接将其放在 jar 包当中 ， 
    这也就导致了我们拉取 代码后出现编译错误。 
    那么为了画过编译，我们要把缺失的jar补回来。

2、问题解决：
    l. 缺失jar引人，如图 1-14所示。
    2. 新增jar在Gradle中生效，如图 1-15所示。
        因为整个 Spring都在 Gradle环境中，
        所以要使得 jar生效就必须更改 Gradle配置文件: compile fileTree(dir: ’libs’ ,include :’* jar')
{% endhighlight %} 
    
##### **cglib 和 obienesis 的编译错误解决**
{% highlight text %}
1、问题描述：
    为了避免第三方 class的冲突， Spring把最新的 cglib和 obj巳nesis给重新打包( repack)了， 
    它并没有在源码里提供这部分的代码，而是直接将其放在 jar 包当中 ， 
    这也就导致了我们拉取 代码后出现编译错误。 
    那么为了画过编译，我们要把缺失的jar补回来。

2、问题解决：
    l. 缺失jar引人。
    2. 新增jar在Gradle中生效。
        因为整个 Spring都在 Gradle环境中，
        所以要使得 jar生效就必须更改 Gradle配置文件: compile fileTree(dir:'libs' ,include :'* jar')
        
        注：spring-core项目 spring-core.gradle配置 添加：compile fileTree(dir: ’libs’ ,include :’* jar')
    
    3、编译缺少spring-cglib-repack-3.2.6.jar、spring-objenesis-repack-2.6.jar
        cd ../../spring-core-5.0.3.RELEASE
        执行jar 
{% endhighlight %} 

##### **Aspecj 编译问题解决**
{% highlight text %}
1、问题描述：
    当完成以上的 jar 包导人工作并进行重新编译后，发现还是有编译错误提醒，真是山路十
    八弯。 查看编译错误原因，发现居然是类找不到。
    **aspect 关键字 Java 语法违背**
    可是我们明明能看到对应的类就在工程里面，为什么会找不到呢?
    发现类的声明居然使用了 aspect而不是 class
2、问题原因：
    **AspectJ 实现 AOP**
    public aspect TxAspect {...}
    
    aspect关键字并不是Java新定义的，只是AspectJ定义的关键字
    
    需要使用 ajc.exe来执行编译:ajc HelloWorld.java TxAspect.java
    
    注：ajc 可以理解为javac ajc可以识别AspectJ定义的关键字
    
3、问题解决：
    3.1. 下载 AspectJ 的最新稳定版本
        安装 AspectJ 之前，请确保系统已经安装了 JDK。 下载下来后是一个jar包
    3.2. AspectJ 安装
        打开命令行， cd 到该 jar包所在的文件夹，运行 java -jar aspectj-1.9.0.jar命令，
        打开 AspectJ 的安装界面。 第一个界面是欢迎界面，直接点击 Next，
        第二个界面中选择 jre 的安装路径，继续点击 Next,
        第三个界面中选择 AspectJ 的安装路径，点击 Install。 
        因为安装过程的实 质是解j王一个压缩包，
        并不需要太多地依赖于系统，因此路径可以任意选择，这里选择和 Java 安装在一起。
{% endhighlight %} 

<center><b><h3>SpringFramework Demo搭建(IntelliJ IDEA详细)</h3></b></center>
#### **SpringFramework Demo项目**       
{% highlight text %}
1、打开Idea -> 创建Maven -> 添加依赖的jar包,如图1.1
    1.1 添加依赖: 
        jar:junit:4.12、commons-logging:1.2
        源码:spring-core-main、spring-beans-main、spring-context-main、spring-expression-main
    
2、创建测试POJO - MyBean.java
3、创建Bean配置文件 - bean_demo.xml
4、创建测试用例 - SpringDemoTest.java
{% endhighlight %}

![图1.1]({{ site.url }}{{site.baseurl}}/assets/img/spring/WechatIMG270.png)

##### **MyBean.java**
{% highlight java %}
public class MyBean {

	private String name = "MyBean 名称";

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}
}
 
{% endhighlight %}


##### **bean_demo.xml**
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">


	<bean id="myBean" class="com.springdemo.cn.pojo.MyBean">

	</bean>
</beans>
 
{% endhighlight %}

##### **SpringDemoTest.java**
{% highlight java %}
    @Test
	public void testBeanFactory() {
		BeanFactory bf = new XmlBeanFactory(new ClassPathResource("bean_demo.xml"));
		MyBean myBean = (MyBean)bf.getBean("myBean");
		System.out.println("Bean的名称:"+myBean.getName());
	}
 
{% endhighlight %}