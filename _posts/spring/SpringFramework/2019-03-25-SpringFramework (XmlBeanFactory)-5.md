---
layout: post
title:  "Spring源码阅读[5]"
date:   2019-3-25 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[5]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>Spring容器基础-XMLBeanFactory</h3></b></center>

### **XmlBeanFactory** 
{% highlight text %}
    Demo代码:
        BeanFactory bf = new XmlBeanFactory(new ClassPathResource("bean_demo.xml"));
    运行时序图如下：
        时序图从BeanFactoryTest测试类开始，通过时序图我们可以一目了然地看到整个逻辑处
    理顺序。 在测试的 BeanFactoryTest 中首先调用 ClassPathResource 的构造函数来构造 Resource 资源文件的实例对象，
    这样后续的资源处理就可以用 Resource提供的各种服务来操作了，当我们有了 Resource 后就可以进行 XmlBeanFactory 的初始化了 。
{% endhighlight %} 
![XMLBeanFactory初始化时序图]({{ site.url }}{{site.baseurl}}/assets/img/spring/XmlBeanFactory初始化时序图.png)
{% highlight text %}
    问题:Resource的配置文件是如何封装的呢？
    解析:
        Resource通过ClassPathResource进行封装；
        ClassPathResource具体完成了什么功能？
            在Java程式，将不同来源的资源抽象成URL，通过注册不同的handler(java.net.URLStreamHandler)处理不同来源的资源读取逻辑，
        一般Handler类型使用了不同的协议(Protocol)来识别，如: "file:"、"http:"、"jar:"、"war:"等，URL没有默认定义相对于 ClassPath
        或ServletContext等识别资源的Handler，虽然可以自定义URLStreamHandler来解析特定的URL协议，比如: "classpath:",
        然而需要深入了解URL的实现机制，且URL也没有提供基本的方法，比如查询当前资源是否存在、检查当前资源是否可读等，
            Spring在自己内部使用Resource实现了自定义Handler来识别自定义协议: Resource接口
        代码如下：
{% endhighlight %}
{% highlight java %}
/**
 * 配置资源读入流简单定义
 */
public interface InputStreamSource {
	InputStream getInputStream() throws IOException;
}
/**
 * Spring自定义协议配置文件处理，如:"classpath:"
 */
public interface Resource extends InputStreamSource {

	/**
	 * 判断配置资源文件是否存储在物理介质上
	 */
	boolean exists();

	/**
	 * 指示是否可以通过{@link #getInputStream（）}读取此资源的内容。
	 */
	default boolean isReadable() {
		return true;
	}

	/**
	 * 指定资源是否打开流读取
     * 如果返回true，则无法通过InputStream多次读取，
     * 读取后必须关闭流，防止资源泄漏
	 */
	default boolean isOpen() {
		return false;
	}

	/**
	 * 确定配置文件是否为文件系统中的文件
	 */
	default boolean isFile() {
		return false;
	}

	/**
	 * 返回配置资源的URL
	 */
	URL getURL() throws IOException;

	/**
	 * 返回配置资源的URI
	 
	 URI,URL,URN:
	    定义: [URI](https://tools.ietf.org/html/rfc3986)
	    
	        “A Uniform Resource Identifier (URI) 是一个紧凑的字符串用来标示抽象或物理资源。”
            
            “A URI 可以进一步被分为定位符、名字或两者都是. 术语“Uniform Resource Locator” (URL) 是URI的子集, 除了确定一个资源,还提供一种定位该资源的主要访问机制(如其网络“位置”)。“
            
            那我们无所不知的维基百科把这段消化的很好，并描述的更加形象了：
            
            “URI可以分为URL,URN或同时具备locators 和names特性的一个东西。URN作用就好像一个人的名字，URL就像一个人的地址。换句话说：URN确定了东西的身份，URL提供了找到它的方式。”
	 
	    区别：
	        1、所有的URL都属于URI，而URI不一定是URL
	        2、URL是提供了"访问机制"、"网络位置"的路径
	        3、URN是URI的一种，用特定命名空间的名字标识资源,使用URN可以在不知道其网络位置及访问方式的情况下讨论资源
	        
        举例：http://bitpoetry.io/posts/hello.html#intro
            属于URI
            http:// 定义访问资源的方式(协议)
            bitpoetry.io/posts/hello.html  资源存放的位置
            
            URL是URI的子集，告诉我们访问网络位置的方式; 如：http://bitpoetry.io/posts/hello.html
            URN是URI的子集，包括名字(给定的命名空间内)，但是不包括访问方式; 如:bitpoetry.io/posts/hello.html#intro
	 */
	URI getURI() throws IOException;

	/**
	 * 获取配置文件File对象
	 */
	File getFile() throws IOException;

	/**
	 * 基于NIO的ReadableByteChannel读取文件信息
	 */
	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
	 * 配置文件内容大小
	 */
	long contentLength() throws IOException;

	/**
	 * 配置文件最后修改的时间
	 */
	long lastModified() throws IOException;

	/**
	 * 在项目相对路径创建配置资源文件
	 */
	Resource createRelative(String relativePath) throws IOException;

	/**
	 * 获取文件名称。及获取路径最后的部分，如："myfile.txt"
	 */
	@Nullable
	String getFilename();

	/**
	 * 返回此资源的描述，
     * 在使用资源时用于错误输出。
	 */
	String getDescription();

}
{% endhighlight %}
{% highlight text %}
    InputStreamSource基础封装返回 InputStream 的方法，如：File、ClassPath下的资源和ByteArray等，
    方法:getInputStream() 获取文件输入流；
    
    Resource接口抽象了所有Spring内部使用的底层资源操作，File、URL、ClassPath等
        1、定义判断当前配置资源状态的方法:存在性(exists)、可读性(isReadable)、是否处于打开状态(isOpen)
        2、Resource接口还提供了不同资源的URL、URI、File类型转换，以及读取文件属性信息。
        3、对不同协议资源文件都有相应的实现: 文件File类(FileSystemResource)、ClassPath资源(ClassPathResource)、
            URL资源(URLResource)、InputStream 资源(InputStreamResource)、Byte数组(ByteArrayResource)等；
            请参照配置文件处理相关类，如下图:
{% endhighlight %}
![资源文件处理相关类图]({{ site.url }}{{site.baseurl}}/assets/img/spring/资源文件处理相关类.png)



             
            
 
