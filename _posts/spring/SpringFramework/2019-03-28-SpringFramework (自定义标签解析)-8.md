---
layout: post
title:  "Spring源码阅读[8]"
date:   2019-3-28 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[8]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>AbstractBeanDefinition</h3></b></center>

### **自定义标签**

{% highlight text %}
    标签解析方法:org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions
    -> 自定义标签解析方法:org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseCustomElement(org.w3c.dom.Element)
{% endhighlight %}


```java
    /**
	 * 解析根元素(beans):"import", "alias", "bean"
	 * Parse the elements at the root level in the document:
	 * "import", "alias", "bean".
	 * @param root the DOM root element of the document
	 */
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		// 判断是否为默认的Bean的命名空间
		if (delegate.isDefaultNamespace(root)) {
			// 获取根元素的子节点
			NodeList nl = root.getChildNodes();
			// 循环处理子节点
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				// 校验Node是否属于Element(标签类)对象
				if (node instanceof Element) {
					Element ele = (Element) node;
					// 如果Bean命名空间定义此Element(标签)
					if (delegate.isDefaultNamespace(ele)) {
						// 解析标签("import", "alias", "bean")
						parseDefaultElement(ele, delegate);
					}
					else {
						// 解析自定义标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			// 解析自定义标签
			delegate.parseCustomElement(root);
		}
	}
```

### **自定义标签使用** 
{% highlight text %}
    解决自定义标签的途径:
    方式一:使用原生态方式去解析XML文件，然后转化成配置对象；
    方式二(折中方案):使用spring提供的Schema扩展；
        扩展自定义标签的步骤如下:
        1、创建一个需要扩展的组件。
        2、定义一个 XSD 文件描述组件内容。
        3、创建一个文件，实现 BeanDefinitionParser 接口，用来解析 XSD 文件中的定义和组件定义。
        4、创建一个 Handler 文件，扩展自 NamespaceHandlerSupport，目的是将组件注册到 Spring 容器。
        5、编写 Spring.handlers 和 Spring.schemas 文件 。
    
{% endhighlight %}

> 下面是实现自定义标签的过程(动手编码):
> <br/> 1、创建一个POJO类，用来接收配置文件

```java
/**
 * 定义POJO类，用于接收配置文件属性
 */
public class User {

	private String userName;
	private String email;


	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getEmail() {
		return email;
	}

	public void setEmail(String email) {
		this.email = email;
	}
}
```

> 2、定义一个 XSD文件描述组件内容. 例如:user_schema.xsd 

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<xsd:schema xmlns="http://www.oco51.com/schema/user_schema"
			xmlns:xsd="http://www.w3.org/2001/XMLSchema"
			targetNamespace="http://www.oco51.com/schema/user_schema"
			elementFormDefault="qualified">


	<xsd:element name="user">
		<xsd:complexType>
			<!-- 注册Bean的主键 -->
			<xsd:attribute name="id" type="xsd:string" />
			<xsd:attribute name="userName" type="xsd:string" />
			<xsd:attribute name="email" type="xsd:string" />
		</xsd:complexType>
	</xsd:element>

</xsd:schema>
```

> 3、创建一个文件 ，实现 BeanDefinitionParser接口，用来解析 XSD 文件中的定义和组件定义。

```java
import com.springdemo.cn.custom_element.pojo.User;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.xml.AbstractSimpleBeanDefinitionParser;
import org.springframework.util.StringUtils;
import org.w3c.dom.Element;

/**
 * 解析自定义Schema文件
 */
public class UserBeanDefinitionParser extends AbstractSimpleBeanDefinitionParser {

	/** Element对应的解析类 */
	@Override
	protected Class getBeanClass(Element element) {
		return User.class;
	}

	/** 从Schema中Element解析并提取对应的元素 */
	@Override
	protected void doParse(Element element, BeanDefinitionBuilder builder) {
		// 提取标签属性
		String userName = element.getAttribute("userName");
		String email = element.getAttribute("email");

		// 将提取的数据放入BeanDefinitionBuilder中，待到完成所有的Bean解析后统一注册到BeanFactory中
		if (StringUtils.hasText(userName)) {
			builder.addPropertyValue("userName", userName);
		}
		if (StringUtils.hasText(email)) {
			builder.addPropertyValue("email", email);
		}
	}
}
```

> 4、创建一个 Handler 文件，扩展 自 NamespaceHandlerSupport，目的是将组件注册到 Spring 容器 。

```java
import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

/**
 * 将自定义主键注册到Spring容器中
 */
public class UserNamespaceHandler extends NamespaceHandlerSupport {
	/**
	 * elementName:XSD中定义的名称
	 */
	@Override
	public void init() {
		registerBeanDefinitionParser("user", new UserBeanDefinitionParser());
	}
}
```

> 5、编写 Spring.handlers 和 Spring.schemas 文件， 默认位置是在工程 的/META-INF/文件夹 下，当然，你可以通过 Spring 的扩展或者修改源码的方式改变路径 。

{% highlight text %}
    在resources目录下新建 /META-INF文件夹，spring默认读取
    新建文件 spring.handlers 和 spring.schemas
    spring.handlers:
    http\://www.oco51.com/schema/user_schema=com.springdemo.cn.custom_element.UserNamespaceHandler
    
    spring.schemas
    http\://www.oco51.com/schema/user_schema.xsd=user_schema.xsd
    
{% endhighlight %}

> 6、创建测试配置文件，在配置文件中引入对应的命名空间以及 XSD 后，便可以直接使用自定义定义标签。 例如:custom_element_demo.xml

{% highlight xml %}
    <?xml version="1.0" encoding="UTF-8"?>
    
    <beans xmlns="http://www.springframework.org/schema/beans"
    	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	   xmlns:ui="http://www.oco51.com/schema/user_schema"
    	   xsi:schemaLocation="http://www.springframework.org/schema/beans
               http://www.springframework.org/schema/beans/spring-beans.xsd
               http://www.oco51.com/schema/user_schema
               http://www.oco51.com/schema/user_schema.xsd
    ">
    
    	<ui:user id="userBean" userName="张志宏测试" email="45@qq.com"/>
    
    </beans>
{% endhighlight %}

> 7、新建测试类

{% highlight java %}
    import com.springdemo.cn.custom_element.pojo.User;
    import org.springframework.beans.factory.BeanFactory;
    import org.springframework.beans.factory.xml.XmlBeanFactory;
    import org.springframework.context.ApplicationContext;
    import org.springframework.context.support.ClassPathXmlApplicationContext;
    import org.springframework.core.io.ClassPathResource;
    
    /**
     * 自定义标签测试
     */
    public class CustomElementTest {
    
    	
    	public static void main(String[] args) {
    	    // 方式一
    		BeanFactory bf = new XmlBeanFactory(new ClassPathResource("custom_element_demo.xml"));
    		User user1 = (User)bf.getBean("userBean");
    		System.out.println(user1.getUserName());
    		System.out.println(user1.getEmail());
    
    		System.out.println("==================");
            
            // 方式二
    		ApplicationContext context = new ClassPathXmlApplicationContext("custom_element_demo.xml");
    		User user2 = (User)context.getBean("userBean");
    		System.out.println(user2.getUserName());
    		System.out.println(user2.getEmail());
    	}
    
    
    }
{% endhighlight %}

> 以上是通过测试用例来认识自定义标签

### **自定义标签解析**

> Spring的入口方法:org.springframework.context.support.AbstractApplicationContext#refresh(通过跟踪ClassPathXmlApplicationContext获取)

> <br/>探究下自定义标签的解析过程：
> <br/> 解析步骤大致如下:
> <br/> 1、根据对应的bean获取标签的命名空间
> <br/> 2、通过命名空间解析对应的处理器
> <br/> 3、根据用户自定义的处理器解析

{% highlight java %}
    @Nullable
    public BeanDefinition parseCustomElement(Element ele) {
        return parseCustomElement(ele, null);
    }

    @Nullable
    public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        // 获取解析名称空间定义
        String namespaceUri = getNamespaceURI(ele);
        if (namespaceUri == null) {
            return null;
        }
        // 封装指定的namespaceUri到NamespaceHandler
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        }
        // 返回解析的BeanDefinition
        return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    }
{% endhighlight %}

> **1、获取标签的命名空间**
> <br/> 通过org.w3c.dom.Node提供的getNamespaceURI()方法直接获取,例如上面的测试用例:自定义命名空间:http://www.oco51.com/schema/user_schema

{% highlight java%}
@Nullable
public String getNamespaceURI(Node node) {
    return node.getNamespaceURI();
}
{% endhighlight %}

> **2、提取自定义标签处理器**
> <br/> 通过自定义命名空间，提取NameSpaceHandler
> <br/> 提取方法: NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
> <br/> 跟踪提取: (XmlReaderContext)readerContext在初始化的时候，属性namespaceHandlerResolver已经被初始化为NamespaceHandlerResolver，
> <br/> 调用NamespaceHandlerResolver中的resolve方法获取NamespaceHandler
> <br/> 调用PluggableSchemaResolver中的resolveEntity解析PluggableSchema

> <br/> 解析自定义NamespaceHandler(spring.handles配置):org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver#resolve
```java
    @Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
		// 获取已配置的所有handler映射：spring.handlers
		Map<String, Object> handlerMappings = getHandlerMappings();
		// 通过命名空间找到对应的Handle或Class
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			// 已经解析过的，从handlerMappings缓存中读取
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			// 没有做过解析的直接返回类路径
			String className = (String) handlerOrClassName;
			try {
				// 通过反射将类路径转换为类实例
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				// 将配置的继承了NamespaceHandlerSupport的类转换成NamespaceHandler
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
				// 初始化
				namespaceHandler.init();
				// 将类写入到缓存 key=namespaceUri;value=namespaceHandler;
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
						"] for namespace [" + namespaceUri + "]", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
						className + "] for namespace [" + namespaceUri + "]", err);
			}
		}
	}
	
	private Map<String, Object> getHandlerMappings() {
        // 初始化值
        // org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions
        // -> documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        // -> createReaderContext(resource)
        // -> 默认加载META-INF/spring.handlers
        Map<String, Object> handlerMappings = this.handlerMappings;
        // 初始化的handler缓存为空
        if (handlerMappings == null) {
            synchronized (this) {
                handlerMappings = this.handlerMappings;
                if (handlerMappings == null) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
                    }
                    try {
                        // 重新加载配置获取handlerMappings
                        Properties mappings =
                                PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
                        if (logger.isDebugEnabled()) {
                            logger.debug("Loaded NamespaceHandler mappings: " + mappings);
                        }
                        handlerMappings = new ConcurrentHashMap<>(mappings.size());
                        // 合并
                        CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
                        // 写入缓存
                        this.handlerMappings = handlerMappings;
                    }
                    catch (IOException ex) {
                        throw new IllegalStateException(
                                "Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
                    }
                }
            }
        }
        return handlerMappings;
    }
```

{% highlight text%}
    步骤1.解析自定义BeanDefinition(继承 AbstractSimpleBeanDefinitionParser);
    步骤2.调用NamespaceHandler.init(),初始化自定义标签，例如:registerBeanDefinitionParser("user", new UserBeanDefinitionParser());
    步骤3.注册完成后,解析器就可以根据自定义的标签进行解析;
        读取配置文件:PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
        配置文件路径:classpath路径下META-INF/spring.handlers
        自定义标签解析:handler.parse(ele, new ParserContext(this.readerContext, this, containingBd))
{% endhighlight %}

### **标签解析详解**
{% highlight text%}
    通过上面的步骤大致了解了程序是如何加载我们自定义标签的BeanDefinition，接下来是通过XSD定义解析标签:
    首先handler.parse(ele, new ParserContext(this.readerContext, this, containingBd))方法并没有在我们自定义的标签解析类中，
    我们自定义标签解析器中定义的是doParse方法。
    其实我们可以在父类NamespaceHandlerSupport中找到parse方法的实现，解析器解析的具体实现AbstractBeanDefinitionParser.parse方法
    注：标签定义中尽量避免使用id、name做属性，因为解析方法通过id或name加载Bean
{% endhighlight %}


{% highlight java%}
    @Override
    @Nullable
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        // 查找解析器
        BeanDefinitionParser parser = findParserForElement(element, parserContext);
        // 使用解析器进行解析操作
        return (parser != null ? parser.parse(element, parserContext) : null);
    }
{% endhighlight %}

> <br/> 获取自定义标签解析器过程

{% highlight java%}
    @Nullable
    private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
        // 获取自定义解析器
        // 例如:<ui:user id="userBean" userName="..." email="..."/> user就是自定义标签的key
        String localName = parserContext.getDelegate().getLocalName(element);
        // 通过自定义解析器配置:registerBeanDefinitionParser("user", new UserBeanDefinitionParser());获取自定义解析器
        BeanDefinitionParser parser = this.parsers.get(localName);
        if (parser == null) {
            parserContext.getReaderContext().fatal(
                    "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
        }
        return parser;
    }
{% endhighlight %}
            
> <br/> [核心]解析自定义标签 AbstractBeanDefinitionParser.parse

{% highlight java%}
    
{% endhighlight %}            
            
            
            
            
 
