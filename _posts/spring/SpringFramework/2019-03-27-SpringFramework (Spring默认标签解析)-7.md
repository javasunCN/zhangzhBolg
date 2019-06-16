---
layout: post
title:  "Spring源码阅读[7]"
date:   2019-3-27 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[7]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>Spring默认标签解析</h3></b></center>

### **Spring Bean声明** 
{% highlight text %}
    Spring的XML配置文件中，Bean的声明方式:
    一、默认<bean>标签声明：
        例如:<bean id="myBean" class="xxx.xxx.MyBean"/>
    二、自定义标签声明:
        例如:<tx:annotation-driven/>
        
    
{% endhighlight %}

{% highlight text %}
    > 下面是对默认标签解析的源码解读:
    >   核心入口方法:org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement
    >   处理的标签: "import"、"alias"、"bean"、"beans"
    >   处理方式:
    >       1、"import": DefaultBeanDefinitionDocumentReader.importBeanDefinitionResource(Element);
    >       2、"alias":  DefaultBeanDefinitionDocumentReader.processAliasRegistration(Element);
    >       3、"bean":   DefaultBeanDefinitionDocumentReader.processBeanDefinition(Element, BeanDefinitionParserDelegate);
    >       4、"beans":  递归调用 -> DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions(Element)
        
    
{% endhighlight %}

#### **bean的标签解析注册(核心)**
> Bean的解析和注册最复杂、最重要，重点分析

```java
 	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
 		// 解析<bean>标签中的每个属性，例如 class、name、id、alias 之类的属性。
 		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
 		if (bdHolder != null) {
 			// 解析自定义属性
 			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
 			try {
 				// Register the final decorated instance.
 				// 委托BeanDefinitionReaderUtils注册Bean
 				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
 			}
 			catch (BeanDefinitionStoreException ex) {
 				getReaderContext().error("Failed to register bean definition with name '" +
 						bdHolder.getBeanName() + "'", ele, ex);
 			}
 			// Send registration event.
 			// 通知监听器Bean加载完成
 			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
 		}
 	}
```

> Bean解析：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element)

```java
    @Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    
        // 1、解析id、name属性
        String id = ele.getAttribute(ID_ATTRIBUTE);
        String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

        List<String> aliases = new ArrayList<>();
        if (StringUtils.hasLength(nameAttr)) {
            // 2、分割name属性
            String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            aliases.addAll(Arrays.asList(nameArr));
        }

        String beanName = id;
        if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
            // 从别名列表中获取bean的名称
            beanName = aliases.remove(0);
            if (logger.isDebugEnabled()) {
                logger.debug("No XML 'id' specified - using '" + beanName +
                        "' as bean name and " + aliases + " as aliases");
            }
        }

        if (containingBean == null) {
            checkNameUniqueness(beanName, aliases, ele);
        }

        // 3、 解析属性， 封装到 GenericBeanDefinition 类中
        AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
        if (beanDefinition != null) {
            // 4、没有指定beanName，按照Spring中提供的命名规则生成beanName
            if (!StringUtils.hasText(beanName)) {
                try {
                    if (containingBean != null) {
                        beanName = BeanDefinitionReaderUtils.generateBeanName(
                                beanDefinition, this.readerContext.getRegistry(), true);
                    }
                    else {
                        beanName = this.readerContext.generateBeanName(beanDefinition);
                        // Register an alias for the plain bean class name, if still possible,
                        // if the generator returned the class name plus a suffix.
                        // This is expected for Spring 1.2/2.0 backwards compatibility.
                        String beanClassName = beanDefinition.getBeanClassName();
                        if (beanClassName != null &&
                                beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                                !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                            aliases.add(beanClassName);
                        }
                    }
                    if (logger.isDebugEnabled()) {
                        logger.debug("Neither XML 'id' nor 'name' specified - " +
                                "using generated bean name [" + beanName + "]");
                    }
                }
                catch (Exception ex) {
                    error(ex.getMessage(), ele);
                    return null;
                }
            }
            String[] aliasesArray = StringUtils.toStringArray(aliases);
            // 5、将beanDefinition封装到BeanDefinitionHolder
            return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
        }

        return null;
    }
```

{% highlight text %}
    上面方法解析bean标签的大致工作:
    1、获取bean标签属性id、name;
    2、进一步解析bean标签的自定义属性，封装到GenericBeanDefinition实例中;
    3、如果没指定beanName，使用Spring默认规则生成beanName(BeanDefinitionReaderUtils.generateBeanName);
    4、将解析的beanDefinition封装到BeanDefinitionHolder
    
{% endhighlight %}
  
> 解析bean标签的所有属性，封装到GenericBeanDefinition实例中:
>   org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element, java.lang.String, org.springframework.beans.factory.config.BeanDefinition)          

```java
	@Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		// 解析class属性
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		// 解析parent属性
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
			// GenericBeanDefinition extends AbstractBeanDefinition
			// 创建承载属性的AbstractBeanDefinition类型GenericBeanDefinition对象
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			// 硬编码解析Bean的各种属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			// 获取description子标签
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			// 解析元数据
			parseMetaElements(ele, bd);
			// 解析lookup-method属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			// 解析replaced-method属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			// 解析constructor-arg(构造函数参数)属性
			parseConstructorArgElements(ele, bd);
			// 解析property属性
			parsePropertyElements(ele, bd);
			// 解析qualifier属性
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

{% highlight text %}
    1、创建属性承载对象 BeanDefinition
        BeanDefinition接口的三种实现:
            1、RootBeanDefinition
            2、ChildBeanDefinition
            3、GenericBeanDefinition
        这三个实现类都继承AbstractBeanDefinition，
        BeanDefinition是<bean>元素标签在容器内部对应的实现类，<bean>元素标签拥有的属性与BeanDefinition的属性是一一对应，
        RootBeanDefinition：解析父<bean>
        ChildBeanDefinition：解析子<bean>
        GenericBeanDefinition：Spring2.5加入 用于标准bean定义目的的一站式服务。可以定义parentName来灵活的配置父Bean，父Bean解析后会覆盖RootBeanDefinition解析的Bean
        
        BeanDefinition将<bean>标签转换为容器内部表示后，BeanDefinition将Bean信息注册到BeanDefinitionRegistry(相当于Bean的内存数据库，Map存储)，
        后续Spring从BeanDefinitionRegistry读取Bean的配置信息.
        
        // 首先创建GenericBeanDefinition，承载Bean的属性信息
        protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
        			throws ClassNotFoundException {
        
            return BeanDefinitionReaderUtils.createBeanDefinition(
                    parentName, className, this.readerContext.getBeanClassLoader());
        }
        // BeanDefinitionReaderUtils.createBeanDefinition
        public static AbstractBeanDefinition createBeanDefinition(
        			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {
        
            GenericBeanDefinition bd = new GenericBeanDefinition();
            bd.setParentName(parentName);
            if (className != null) {
                // 如果传入的类加载器不为null，则实例化Bean，如果为空，只记录Bean名称
                if (classLoader != null) {
                    //立即实例化Bean
                    bd.setBeanClass(ClassUtils.forName(className, classLoader));
                }
                else {
                    bd.setBeanClassName(className);
                }
            }
            // 使用父类名+类名 生成GenericBeanDefinition
            return bd;
        }
    
{% endhighlight %}

> **bean标签属性**

| **标签** | **说明** | 
|:--------|:-------:|
| id   | Bean的唯一标识名。它必须是合法的XML ID，在整个XML文档中唯一   |
| name   | 用来为id创建一个或多个别名。它可以是任意的字母符合。多个别名之间用逗号或空格分开   |
| class   | 用来定义类的全限定名（包名＋类名）。只有子类Bean不用定义该属性   |
| parent   | 子类Bean定义它所引用它的父类Bean。这时前面的class属性失效。子类Bean会继承父类Bean的所有属性，子类Bean也可以覆盖父类Bean的属性。注意：子类Bean和父类Bean是同一个Java类   |
| abstract   | 默认为”false”，用来定义Bean是否为抽象Bean。它表示这个Bean将不会被实例化，一般用于父类Bean，因为父类Bean主要是供子类Bean继承使用   |
| singleton   | 默认为“true”，定义Bean是否是Singleton（单例）。如果设为“true”,则在BeanFactory作用范围内，只维护此Bean的一个实例。如果设为“flase”，Bean将是Prototype（原型）状态，BeanFactory将为每次Bean请求创建一个新的Bean实例   |
| lazy-init   | 默认为“default”，用来定义这个Bean是否实现懒初始化。如果为“true”，它将在BeanFactory启动时初始化所有的Singleton Bean。反之，如果为“false”,它只在Bean请求时才开始创建Singleton Bean   |
| autowire   | 自动装配，默认为“default”，它定义了Bean的自动装载方式：1、“no”:不使用自动装配功能。2、“byName”:通过Bean的属性名实现自动装配。3、“byType”:通过Bean的类型实现自动装配。4、“constructor”:类似于byType，但它是用于构造函数的参数的自动组装。5、“autodetect”:通过Bean类的反省机制（introspection）决定是使用“constructor”还是使用“byType”。   |
| dependency-check   | 依赖检查，默认为“default”，它用来确保Bean组件通过JavaBean描述的所以依赖关系都得到满足。在与自动装配功能一起使用时，它特别有用。1、 none：不进行依赖检查。2、 objects：只做对象间依赖的检查。3、 simple：只做原始类型和String类型依赖的检查。4、 all：对所有类型的依赖进行检查。它包括了前面的objects和simple。|
| depends-on   | 依赖对象，这个Bean在初始化时依赖的对象，这个对象会在这个Bean初始化之前创建。   |
| init-method   | 用来定义Bean的初始化方法，它会在Bean组装之后调用。它必须是一个无参数的方法。   |
| destroy-method   | 用来定义Bean的销毁方法，它在BeanFactory关闭时调用。同样，它也必须是一个无参数的方法。它只能应用于singleton Bean。   |
| factory-method   | 定义创建该Bean对象的工厂方法。它用于下面的“factory-bean”，表示这个Bean是通过工厂方法创建。此时，“class”属性失效。   |
| factory-bean   | 定义创建该Bean对象的工厂类。如果使用了“factory-bean”则“class”属性失效。   |
| lookup-method   | 非单例bean加载。使用spring动态代理(CGLIB)加载   |
{: rules="groups"}

> 针对bean的标签属性，拆分成不同的方法进行解析
> <br/>解析标签：scope、abstract、lazy-init、autowire、depends-on、autowire-candidate、primary、init-method、destroy-method
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionAttributes

```java
    public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {

		// singleton spring旧版本使用，新版本:scope
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
		// 解析scope属性
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.
			// 获取默认scope
			bd.setScope(containingBean.getScope());
		}

		// 解析abstract属性
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}

		// 解析 lazy-init 属性
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (isDefaultValue(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

		// 解析 autowire 属性
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));

		// 解析 depends-on 属性
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}

		// 解析 autowire-candidate 属性
		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if (isDefaultValue(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}

		// 解析 primary 属性
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

		// 解析 init-method 属性
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}

		// 解析 destroy-method 属性
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}

		// 解析 factory-method 属性
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
		// 解析 factory-bean 属性
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}

```

> 解析标签： description
> <br/>解析方法： DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT)

> 解析标签：meta(元数据)
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseMetaElements

```xml
<bean id="myBean" class="beanClass">
    <!-- 设置变量值 -->
    <meta key="testStr" value="aaaaaaaa"/>
</bean>
```

```java 
    public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
		// 获取子节点属性
		NodeList nl = ele.getChildNodes();
		// 循环处理元数据标签(meta)信息
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
				Element metaElement = (Element) node;
				String key = metaElement.getAttribute(KEY_ATTRIBUTE);
				String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
				BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
				attribute.setSource(extractSource(metaElement));
				attributeAccessor.addMetadataAttribute(attribute);
			}
		}
	}
```

> 解析标签：lookup-method
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseLookupOverrideSubElements
> <br/>标签释义：获取器注入，获取器注入是一种特殊的方法注入，它是把一个方法声明为返回某种类型的 bean，但实际要返回的 bean 是在配置文件里面配置信息，此方法可用在设计有些可插拔的功能上，解除程序依赖。

```xml
<!-- 这是2个非单例模式的bean -->
<bean id="myBean" class="xxx.xxx.MyBean" scope="prototype"/>

<bean class="beanClass">
    <!-- name:方法 bean:non-singleton-bean -->
    <lookup-method name="method" bean="myBean"/>
</bean>
```

```java
    public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			// 解析 lookup-method 属性
			if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
				Element ele = (Element) node;
				String methodName = ele.getAttribute(NAME_ATTRIBUTE);
				String beanRef = ele.getAttribute(BEAN_ELEMENT);
				LookupOverride override = new LookupOverride(methodName, beanRef);
				override.setSource(extractSource(ele));
				overrides.addOverride(override);
			}
		}
	}
```

> lookup-method示例如下：

```java
// User.java
public class User {

	public void eat() {
		System.out.println("User 吃东西");
	}
}
// Teacher.java
public class Teacher extends User {

	@Override
	public void eat() {
		System.out.println("Teacher 吃东西");
	}
}
// GetBean.java
public abstract class GetBean {

	public void eat() {
		getBean().eat();
	}

	public abstract User getBean();
}

// 测试类 LookupLabelTest.java
public class LookupLabelTest {

	@Test
	public void testLookupMethod() {
		BeanFactory bf = new XmlBeanFactory(new ClassPathResource("bean_demo.xml"));
		GetBean getBean = (GetBean)bf.getBean("getTestBean");

		getBean.eat();
	}
}
```

> lookup-method配置

```xml
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">



	<!-- lookup-method 标签测试 -->
	<bean id="getTestBean" class="com.springdemo.cn.bean_definition_label.lookup_method.pojo.GetBean">
		<lookup-method name="getBean" bean="teacher"/>
	</bean>
	<bean id="user" class="com.springdemo.cn.bean_definition_label.lookup_method.pojo.User"/>
	<bean id="teacher" class="com.springdemo.cn.bean_definition_label.lookup_method.pojo.Teacher"/>

</beans>
```

> 解析标签：replaced-method
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseReplacedMethodSubElements
> <br/>标签释义：方法替换:可以在运行时用新的方法替换现有的方法 。 与之前的 look-up 不同的是， replaced-method不但可以动态地替换返回实体 bean，而且还能动态地更改原有方法的逻辑。

```java
    public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) {
				Element replacedMethodEle = (Element) node;
				String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE);
				String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE);
				ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
				// Look for arg-type match elements.
				List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT);
				for (Element argTypeEle : argTypeEles) {
					String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE);
					match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
					if (StringUtils.hasText(match)) {
						replaceOverride.addTypeIdentifier(match);
					}
				}
				replaceOverride.setSource(extractSource(replacedMethodEle));
				overrides.addOverride(replaceOverride);
			}
		}
	}
```

> replaced-method示例如下：

```java
// TestChangeMethod.java
public class TestChangeMethod {

	public void changeMe() {
		System.out.println("TestChangeMethod#changeMe");

	}
}
// TestMethodReplacer.java

import org.springframework.beans.factory.support.MethodReplacer;
import java.lang.reflect.Method;

public class TestMethodReplacer implements MethodReplacer {

	@Override
	public Object reimplement(Object obj, Method method, Object[] args) throws Throwable {
		System.out.println("我替换了原有的方法");
		return null;
	}
}

// 测试类 ReplacedMethodTest.java
public class ReplacedMethodTest {

	@Test
	public void testLookupMethod() {
		BeanFactory bf = new XmlBeanFactory(new ClassPathResource("bean_demo.xml"));
		TestChangeMethod getBean = (TestChangeMethod)bf.getBean("testChangeMethod");

		getBean.changeMe();

	}
}
```

> replaced-method配置

```xml
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">



	<!-- replaced-method 标签测试 -->
    <bean id="testChangeMethod" class="com.springdemo.cn.bean_definition_label.replaced_method.pojo.TestChangeMethod">
        <replaced-method name="changeMe" replacer="testMethodReplacer"/>
    </bean>
    <bean id="testMethodReplacer" class="com.springdemo.cn.bean_definition_label.replaced_method.pojo.TestMethodReplacer"/>

</beans>
```

> 解析标签：构造函数 constructor-arg
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseConstructorArgElements

```java
    // 构造方法解析相对复杂
    public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, CONSTRUCTOR_ARG_ELEMENT)) {
				parseConstructorArgElement((Element) node, bd);
			}
		}
	}
	
	public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
    		// 指定构造参数的顺序(index)
    		String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
    		// 构造参数类型(type)
    		String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
    		// 构造参数名称name
    		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    		if (StringUtils.hasLength(indexAttr)) {
    			try {
    				int index = Integer.parseInt(indexAttr);
    				// 无参构造的参数为0，不可能小于0
    				if (index < 0) {
    					error("'index' cannot be lower than 0", ele);
    				}
    				else {
    					try {
    						this.parseState.push(new ConstructorArgumentEntry(index));
    						// 解析元素对应的属性值
    						Object value = parsePropertyValue(ele, bd, null);
    						ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
    						if (StringUtils.hasLength(typeAttr)) {
    							valueHolder.setType(typeAttr);
    						}
    						if (StringUtils.hasLength(nameAttr)) {
    							valueHolder.setName(nameAttr);
    						}
    						// 封装解析出来的元素
    						valueHolder.setSource(extractSource(ele));
    						// 不能重复的指定相同参数
    						if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
    							error("Ambiguous constructor-arg entries for index " + index, ele);
    						}
    						else {
    							bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
    						}
    					}
    					finally {
    						this.parseState.pop();
    					}
    				}
    			}
    			catch (NumberFormatException ex) {
    				error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
    			}
    		}
    		else {
    			try {
    				// 没有index则忽略，顺序查找
    				this.parseState.push(new ConstructorArgumentEntry());
    				Object value = parsePropertyValue(ele, bd, null);
    				// 封装解析出来的元素
    				ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
    				if (StringUtils.hasLength(typeAttr)) {
    					valueHolder.setType(typeAttr);
    				}
    				if (StringUtils.hasLength(nameAttr)) {
    					valueHolder.setName(nameAttr);
    				}
    				valueHolder.setSource(extractSource(ele));
    				bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
    			}
    			finally {
    				this.parseState.pop();
    			}
    		}
    	}
    	
    	/**
    	获取属性值
    	*/
    	public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
            String elementName = (propertyName != null ?
                    "<property> element for property '" + propertyName + "'" :
                    "<constructor-arg> element");
    
            // Should only have one child element: ref, value, list, etc.
            // 应该只有一个子元素：ref，value，list等。
            NodeList nl = ele.getChildNodes();
            Element subElement = null;
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                // description 和 meta标签不处理
                if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
                        !nodeNameEquals(node, META_ELEMENT)) {
                    // Child element is what we're looking for.
                    if (subElement != null) {
                        error(elementName + " must not contain more than one sub-element", ele);
                    }
                    else {
                        subElement = (Element) node;
                    }
                }
            }
    
            // 判断 constructor-arg 上的 ref 属性
            boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
            // 判断 constructor-arg 上的 value 属性
            boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
            /*
                在 constructor-arg 不存在 .
                1. 同时既有ref属性又有value属性
                2. 存在 ref 属性或者 value 属性且又有子元素
            */
            if ((hasRefAttribute && hasValueAttribute) ||
                    ((hasRefAttribute || hasValueAttribute) && subElement != null)) {
                error(elementName +
                        " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
            }
            // 处理 ref 属性
            if (hasRefAttribute) {
                String refName = ele.getAttribute(REF_ATTRIBUTE);
                if (!StringUtils.hasText(refName)) {
                    error(elementName + " contains empty 'ref' attribute", ele);
                }
                // 使用 RuntimeBeanReference 封装ref名称
                RuntimeBeanReference ref = new RuntimeBeanReference(refName);
                ref.setSource(extractSource(ele));
                return ref;
            }
            // 处理value属性
            else if (hasValueAttribute) {
                // 使用 TypedStringValue封装 value属性
                TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
                valueHolder.setSource(extractSource(ele));
                return valueHolder;
            }
            else if (subElement != null) {
                // ⑤解析子元素
                return parsePropertySubElement(subElement, bd);
            }
            else {
                // Neither child element nor "ref" or "value" attribute found.
                error(elementName + " must specify a ref or value", ele);
                return null;
            }
        }
        
        // ⑤解析子元素  parsePropertySubElement
        @Nullable
        public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd) {
            return parsePropertySubElement(ele, bd, null);
        }
        /**
         * 解析<property>元素中ref,value或者集合等子元素
         * Parse a value, ref or collection sub-element of a property or
         * constructor-arg element.
         * @param ele subelement of property element; we don't know which yet
         * @param defaultValueType the default type (class name) for any
         * {@code <value>} tag that might be created
         */
        @Nullable
        public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
            // 校验标签是否为默认的命名空间
            if (!isDefaultNamespace(ele)) {
                return parseNestedCustomElement(ele, bd);
            }
            // 如果子元素是bean，则使用解析<Bean>元素的方法解析
            else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
                BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
                if (nestedBd != null) {
                    nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
                }
                return nestedBd;
            }
            // 解析ref标签
            else if (nodeNameEquals(ele, REF_ELEMENT)) {
                // A generic reference to any name of any bean.
                // 获取<property>元素中的bean属性值，引用其他解析的Bean的名称
                String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
                boolean toParent = false;
                if (!StringUtils.hasLength(refName)) {
                    // A reference to the id of another bean in a parent context.
                    // 获取<property>元素中parent属性值，引用父级容器中的Bean
                    refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
                    toParent = true;
                    if (!StringUtils.hasLength(refName)) {
                        error("'bean' or 'parent' is required for <ref> element", ele);
                        return null;
                    }
                }
                if (!StringUtils.hasText(refName)) {
                    error("<ref> element contains empty target attribute", ele);
                    return null;
                }
                // 创建ref类型数据，指向被引用的对象
                RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
                // 设置引用类型值是被当前子元素所引用
                ref.setSource(extractSource(ele));
                return ref;
            }
            // 如果子元素是<idref>，使用解析ref元素的方法解析
            else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
                return parseIdRefElement(ele);
            }
            // 如果子元素是<value>，使用解析value元素的方法解析
            else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
                return parseValueElement(ele, defaultValueType);
            }
            // 如果子元素是null，为<property>设置一个封装null值的字符串数据
            else if (nodeNameEquals(ele, NULL_ELEMENT)) {
                // It's a distinguished null value. Let's wrap it in a TypedStringValue
                // object in order to preserve the source location.
                TypedStringValue nullHolder = new TypedStringValue(null);
                nullHolder.setSource(extractSource(ele));
                return nullHolder;
            }
    
            // 如果子元素是<array>，使用解析array集合子元素的方法解析
            else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
                // 解析<array>集合子元素
                return parseArrayElement(ele, bd);
            }
            // 如果子元素是<list>，使用解析list集合子元素的方法解析
            else if (nodeNameEquals(ele, LIST_ELEMENT)) {
                // 解析<list>集合子元素
                return parseListElement(ele, bd);
            }
            // 如果子元素是<set>，使用解析set集合子元素的方法解析
            else if (nodeNameEquals(ele, SET_ELEMENT)) {
                // 解析<set>集合子元素
                return parseSetElement(ele, bd);
            }
            // 如果子元素是<map>，使用解析map集合子元素的方法解析
            else if (nodeNameEquals(ele, MAP_ELEMENT)) {
                // 解析<map>集合子元素
                return parseMapElement(ele, bd);
            }
            // 如果子元素是<props>，使用解析props集合子元素的方法解析
            else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
                // 解析<props>集合子元素
                return parsePropsElement(ele);
            }
            // 既不是ref，又不是value，也不是集合，则子元素配置错误，返回null
            else {
                error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
                return null;
            }
        }
        
```

{% highlight text %}
    parseConstructorArgElement方法 是解析构造函数配置的参数信息：
    一、提取配置属性(index、type、name)
    二、根据index属性，进行不同的逻辑处理：
        如果配直中指定了 index属性，那么操作步骤如下:
            1. 解析 constructor-arg的子元素。
            2. 使用 ConstructorArgumentValues.ValueHolder类型来封装解析出来的元素。
            3. 将 type、 name 和 index属性一并封装在 ConstructorArgumentValues.ValueHolder类型中
            并添加至当前 BeanDefinition 的 constructorArgumentValues 的 indexedArgumentValues 属性中 。 
        如果没有指定 index 属性，那么操作步骤如下:
            1. 解析 constructor-arg 的子元素 。
            2. 使用 ConstructorArgumentValues.ValueHolder类型来封装解析出来的元素。
            3. 将 type、 name 和 index 属性一并封装在 ConstructorArgurnentValues.ValueHolder类型中
            并添加至当前 BeanDefinition 的 constructorArgumentValues 的 genericArgumentValues 属性中 。
    
    parsePropertyValue方法 获取属性值，方法逻辑：
        1、忽略 description 和 meta标签
        2、提取 constructor-arg 上的 ref 和 value 属性，以便于根据规则验证正确性，其规则为在 constructor-arg 上不存在以下情况 。
          1）、同时既有 ref属性又有 value属性。
          2）、存在 ref属性或者 value 属性且又有子元素。
        3、ref属性处理。使用RuntimeBeanReference封装对应的ref名称。如：<constructor-arg ref="sdaad"/>
        4、value属性处理。使用TypedStringValue封装。如：<constructor-arg value="s"/>
        5、子元素处理，如：
        <constructor-arg>
            <map>
                <entry key="key" value="value" />
            </map> 
        </constructor-arg>
        
    parsePropertySubElement方法解析<property>元素中ref,value或者集合等子元素
        
{% endhighlight %}

> #### 解析子元素 property
> <br/> DTD定义：<!ELEMENT property (description?, meta*, (bean | ref | idref | value | null | list | set | map | props)? )>
> <br/> 具体解析如下:

```java
    /**
	 * 解析<property>标签
	 * Parse a property element.
	 */
	public void parsePropertyElement(Element ele, BeanDefinition bd) {
		// 获取标签的属性 name
		String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
		if (!StringUtils.hasLength(propertyName)) {
			error("Tag 'property' must have a 'name' attribute", ele);
			return;
		}
		// 将name封装成Entry压入栈中
		this.parseState.push(new PropertyEntry(propertyName));
		try {
			// BeanDefinition存在propertyName直接返回
			if (bd.getPropertyValues().contains(propertyName)) {
				error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
				return;
			}
			// 获取属性值
			Object val = parsePropertyValue(ele, bd, propertyName);
			// Spring将 <property>标签封装成PropertyValue
			PropertyValue pv = new PropertyValue(propertyName, val);
			// 解析元数据标签(meta)
			parseMetaElements(ele, pv);
			pv.setSource(extractSource(ele));
			// 将 PropertyValue 添加到 BeanDefinition的MutablePropertyValues属性中
			bd.getPropertyValues().addPropertyValue(pv);
		}
		finally {
			// 从当前的解析栈中弹出该对象
			this.parseState.pop();
		}
	}
```

> 解析标签：property
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parsePropertyElements

```java
    public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
				parsePropertyElement((Element) node, bd);
			}
		}
	}
```

> 解析标签：qualifier
> <br/>解析方法：org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseQualifierElements

```xml
<bean id="testBean" class="xxx.xxx.TestBean">
    <qualifier type="org.springframework.beans.factory.annotation.Qualifier" value="aa"/> 
</bean>
```

```java
    public void parseQualifierElements(Element beanEle, AbstractBeanDefinition bd) {
        NodeList nl = beanEle.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ELEMENT)) {
                parseQualifierElement((Element) node, bd);
            }
        }
    }
    /**
     * Parse a qualifier element.
     *  <qualifier type="org.springframework.beans.factory.annotation.Qualifier" value="a">
     * 		<attribute key="a_key" value="a_value"/>
     * 	</qualifier>
     */
    public void parseQualifierElement(Element ele, AbstractBeanDefinition bd) {
        // 获取标签的 <qualifier> type属性
        String typeName = ele.getAttribute(TYPE_ATTRIBUTE);
        if (!StringUtils.hasLength(typeName)) {
            error("Tag 'qualifier' must have a 'type' attribute", ele);
            return;
        }
        // type压入解析栈中
        this.parseState.push(new QualifierEntry(typeName));
        try {
            AutowireCandidateQualifier qualifier = new AutowireCandidateQualifier(typeName);
            qualifier.setSource(extractSource(ele));
            String value = ele.getAttribute(VALUE_ATTRIBUTE);
            if (StringUtils.hasLength(value)) {
                qualifier.setAttribute(AutowireCandidateQualifier.VALUE_KEY, value);
            }
            NodeList nl = ele.getChildNodes();
            // 循环<qualifier>
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                // 解析 <attribute>
                if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ATTRIBUTE_ELEMENT)) {
                    Element attributeEle = (Element) node;
                    String attributeName = attributeEle.getAttribute(KEY_ATTRIBUTE);
                    String attributeValue = attributeEle.getAttribute(VALUE_ATTRIBUTE);
                    if (StringUtils.hasLength(attributeName) && StringUtils.hasLength(attributeValue)) {
                        // 将key、value封装到BeanMetadataAttribute
                        BeanMetadataAttribute attribute = new BeanMetadataAttribute(attributeName, attributeValue);
                        attribute.setSource(extractSource(attributeEle));
                        // 添加 attribute解析结果添加到AutowireCandidateQualifier
                        qualifier.addMetadataAttribute(attribute);
                    }
                    else {
                        error("Qualifier 'attribute' tag must have a 'name' and 'value'", attributeEle);
                        return;
                    }
                }
            }
            bd.addQualifier(qualifier);
        }
        finally {
            // 从解析栈中弹出
            this.parseState.pop();
        }
    }
```

> #### 总结：
> <br/>综上所述，XML资源文档解析转换成GenericBeanDefinition,IOC可读取的对象，XML中所有配置都可以在GenericBeanDefinition实例类中找到一一对应的属性

> ### AbstractBeanDefinition 通用属性封装

```java
   public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
            implements BeanDefinition, Cloneable {
        // 省略......
        /** 实例化的Bean线程可见 */
        @Nullable
        private volatile Object beanClass;
    
        /** bean的作用范围，对应bean属性scope **/
        @Nullable
        private String scope = SCOPE_DEFAULT;
    
        /** 是否是抽象，默认:false 对应bean属性abstract */
        private boolean abstractFlag = false;
        /** 是否延迟加载，默认:false 对应bean属性lazy-init */
        private boolean lazyInit = false;
        /** 自动注入模式， 默认:不自动注入 对应bean属性autowire */
        private int autowireMode = AUTOWIRE_NO;
    
        /** 依赖检查，Spring 3.0后弃用这个属性 */
        private int dependencyCheck = DEPENDENCY_CHECK_NONE;
    
        /** 用来表示一个bean的实例化依靠另一个bean先实例化，对应bean属性depend-on */
        @Nullable
        private String[] dependsOn;
    
        /**
         * autowire-candidate属性设置为false，这样容器在查找自动装配对象时，
         * 将不考虑该bean，即它不会被考虑作为其他bean自动装配的候选者，
         * 但是该bean本身还是可以使用自动装配来注入其他bean的
         */
        private boolean autowireCandidate = true;
    
        /** 自动装配时出现多个bean候选者时，将作为首选者，对应bean属性primary */
        private boolean primary = false;
        /** 用于记录Qualifier，对应子元素qualifier */
        private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();
    
        @Nullable
        private Supplier<?> instanceSupplier;
    
        /** 允许访问非公开的构造器和方法，程序设置 */
        private boolean nonPublicAccessAllowed = true;
    
        /**
         * 是否以一种宽松的模式解析构造函数，默认为true，
         * 如果为false，则在以下情况
         * interface ITest{}
         * class ITestImpl implements ITest{};
         * class Main {
         *     Main(ITest i) {}
         *     Main(ITestImpl i) {}
         * }
         * 抛出异常，因为Spring无法准确定位哪个构造函数程序设置
         */
        private boolean lenientConstructorResolution = true;
    
        /**
         * 对应bean属性factory-bean，用法：
         * <bean id = "instanceFactoryBean" class = "example.chapter3.InstanceFactoryBean" />
         * <bean id = "currentTime" factory-bean = "instanceFactoryBean" factory-method = "createTime" />
         */
        @Nullable
        private String factoryBeanName;
    
        /**
         * 对应bean属性factory-method
         */
        @Nullable
        private String factoryMethodName;
    
        /**
         * 记录构造函数注入属性，对应bean属性constructor-arg
         */
        @Nullable
        private ConstructorArgumentValues constructorArgumentValues;
    
        /**
         * 普通属性集合
         */
        @Nullable
        private MutablePropertyValues propertyValues;
        /**
         * 方法重写的持有者，记录lookup-method、replaced-method元素
         */
        @Nullable
        private MethodOverrides methodOverrides;
        /**
         * 初始化方法，对应bean属性init-method
         */
        @Nullable
        private String initMethodName;
        /**
         * 销毁方法，对应bean属性destroy-method
         */
        @Nullable
        private String destroyMethodName;
    
        /**
         * 是否执行init-method，默认:是
         */
        private boolean enforceInitMethod = true;
        /**
         * 是否执行destroy-method，默认:是
         */
        private boolean enforceDestroyMethod = true;
        /**
         * 是否是用户定义的而不是应用程序本身定义的，创建AOP时候为true
         */
        private boolean synthetic = false;
    
        /**
         * 定义这个bean的应用，APPLICATION：用户，INFRASTRUCTURE：完全内部使用，与用户无关，
         * SUPPORT：某些复杂配置的一部分
         * 程序设置
         */
        private int role = BeanDefinition.ROLE_APPLICATION;
        /**
         * bean的描述信息
         */
        @Nullable
        private String description;
    
        /**
         * Bean定义资源
         */
        @Nullable
        private Resource resource;
        
        // 省略......
   }
```

> **以上**完成了默认标签的分析和解析，BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
> <br/>**以下**是默认标签中自定义标签的解析函数:bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
> <br/> 在前文提到对bean的解析分两种：一、默认类型解析; 二、自定义类型解析;
> <br/> 那么为什么会在默认类型解析中会加入自定义类型解析？
> <br/> 解：在默认类型解析的时候使用自定义类型解析，其实不是针对Bean解析，而是解析自定类型属性。


```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析<bean>标签中的每个属性，例如 class、name、id、alias 之类的属性。
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 解析自定义标签属性
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // Register the final decorated instance.
            // 委托BeanDefinitionReaderUtils注册Bean
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        // 通知监听器Bean加载完成
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

> 解析默认标签的自定义属性

```java 
    public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder definitionHolder) {
        return decorateBeanDefinitionIfRequired(ele, definitionHolder, null);
    }

    /**
     * 解析自定义标签或bean
     * @param ele 标签元素的自定义属性
     * @param definitionHolder
     * @param containingBd
     * @return
     */
    public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
            Element ele, BeanDefinitionHolder definitionHolder, @Nullable BeanDefinition containingBd) {

        // 忽略对默认标签的解析，因为前面已经解析过了
        BeanDefinitionHolder finalDefinition = definitionHolder;

        // Decorate based on custom attributes first.
        // 获取Element的属性
        NamedNodeMap attributes = ele.getAttributes();
        // 遍历所有属性
        for (int i = 0; i < attributes.getLength(); i++) {
            Node node = attributes.item(i);
            // 使用装饰器封装BeanDefinitionHolder属性
            finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
        }

        // Decorate based on custom nested elements.
        // 遍历所有子节点
        NodeList children = ele.getChildNodes();
        for (int i = 0; i < children.getLength(); i++) {
            Node node = children.item(i);
            if (node.getNodeType() == Node.ELEMENT_NODE) {
                finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
            }
        }
        return finalDefinition;
    }
    public BeanDefinitionHolder decorateIfRequired(
    			Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {
    
        String namespaceUri = getNamespaceURI(node);
        // 复合默认类型解析条件
        if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
            NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
            if (handler != null) {
                // NamespaceHandlerSupport(装饰器模式)装饰Node返回BeanDefinitionHolder
                BeanDefinitionHolder decorated =
                        handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
                if (decorated != null) {
                    return decorated;
                }
            }
            else if (namespaceUri.startsWith("http://www.springframework.org/")) {
                error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
            }
            else {
                // A custom namespace, not to be handled by Spring - maybe "xml:...".
                if (logger.isDebugEnabled()) {
                    logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
                }
            }
        }
        return originalDef;
    }
```

> [核心]注册BeanDefinition
> <br/> BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
> <br/> 注册的方式：一、通过名称注册；二、通过别名注册；

```java 
    /**
	 * 注册Bean到BeanFactory中
	 * Register the given bean definition with the given bean factory.
	 * @param definitionHolder the bean definition including name and aliases
	 * @param registry the bean factory to register with
	 * @throws BeanDefinitionStoreException if registration failed
	 */
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		// 通过名称注册
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		// 通过别名注册
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

> ***注册方式一、通过beanName注册***
> <br/> 注册逻辑：将beanName对应的BeanDefinition直接存储到Map中，key=beanName,value=BeanDefinition

```java
    /**
	 * 注册BeanDefinition
	 * 		缓存结构: Map(BeanName, BeanDefinition)
	 * @param beanName the name of the bean instance to register
	 * @param beanDefinition definition of the bean instance to register
	 * @throws BeanDefinitionStoreException
	 */
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}
		// 判断是否已经注册
		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		// 已注册
		if (existingDefinition != null) {
			// 判断是否允许覆盖
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + existingDefinition + "] bound.");
			}
			//
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (logger.isWarnEnabled()) {
					logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isInfoEnabled()) {
					logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			// 重新覆盖注册的Bean
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		// 未注册
		else {
			// hasBeanCreationStarted()检查Bean是否已创建
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				// 注册的时候防止并发
				synchronized (this.beanDefinitionMap) {
					// 注册Bean
					this.beanDefinitionMap.put(beanName, beanDefinition);
					// 添加beanName到bean的记录表中
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					// bean的注册列表是顺序的、线程可见的
					this.beanDefinitionNames = updatedDefinitions;
					// manualSingletonNames 存放单例Bean的名称，benaName在单例Bean的集合中存在
					if (this.manualSingletonNames.contains(beanName)) {
						// manualSingletonNames新生成一个Set集合
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						// 从新的集合中移除当前加载的beanName
						updatedSingletons.remove(beanName);
						// 替换原来的单例bean集合
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				// 启动注册阶段，beanDefinitionMap加锁
				this.beanDefinitionMap.put(beanName, beanDefinition);
				// 添加到Bean的顺序注册列表
				this.beanDefinitionNames.add(beanName);
				// 从手动注册的单例Bean列表中移除
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			// 重置所有的beanName对应的缓存
			resetBeanDefinition(beanName);
		}
	}
	
	/**
     * 重置给定bean的所有bean定义缓存，
     * 包括从中派生的bean的缓存。
     * Reset all bean definition caches for the given bean,
     * including the caches of beans that are derived from it.
     * @param beanName the name of the bean to reset
     */
    protected void resetBeanDefinition(String beanName) {
        // Remove the merged bean definition for the given bean, if already created.
        // 从 mergedBeanDefinitions 中删除Bean,断开可达链路RootBeanDefinition，以便垃圾回收
        clearMergedBeanDefinition(beanName);

        // Remove corresponding bean from singleton cache, if any. Shouldn't usually
        // be necessary, rather just meant for overriding a context's default beans
        // (e.g. the default StaticMessageSource in a StaticApplicationContext).
        // 从单例缓存中删除相应的bean（如果有）。通常不应该是必要的，而只是用于覆盖上下文的默认bean
        //（例如，StaticApplicationContext中的默认StaticMessageSource）。
        destroySingleton(beanName);

        // Reset all bean definitions that have the given bean as parent (recursively).
        // 递归所有具有指定父级注册的Bean
        for (String bdName : this.beanDefinitionNames) {
            if (!beanName.equals(bdName)) {
                BeanDefinition bd = this.beanDefinitionMap.get(bdName);
                if (beanName.equals(bd.getParentName())) {
                    resetBeanDefinition(bdName);
                }
            }
        }
    }
```

> 以上通过beanName注册Bean的步骤：
> <br/> 1、beanDefinition.validate() 对AbstractBeanDefinition进行校验。与XML校验不同，XML校验的是格式，AbstractBeanDefinition
> 针对的是methodOverrides属性。
> <br/> 2、对已注册的bean判断是否允许覆盖，不允许的话则抛出异常BeanDefinitionStoreException，否则直接覆盖
> <br/> 3、创建状态进行判断：已创建则直接进行注册，创建中的则给bean加锁
> <br/> 4、清除解析之前的bean缓存

> ***注册方式二、通过别名()注册***

```java
    @Override
	public void registerAlias(String name, String alias) {
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
		// 添加Bean别名-加锁
		synchronized (this.aliasMap) {
			// Bean的name与别名相同时，不注册，并删除别名注册的Bean
			if (alias.equals(name)) {
				this.aliasMap.remove(alias);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
				}
			}
			else {
				// 通过别名获取注册的Bean名称
				String registeredName = this.aliasMap.get(alias);
				if (registeredName != null) {
					// 如果别名与bean名称相同时，不注册
					if (registeredName.equals(name)) {
						// An existing alias - no need to re-register
						return;
					}
					// 不允许别名覆盖时，抛出异常IllegalStateException
					if (!allowAliasOverriding()) {
						throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
								name + "': It is already registered for name '" + registeredName + "'.");
					}
					if (logger.isInfoEnabled()) {
						logger.info("Overriding alias '" + alias + "' definition for registered name '" +
								registeredName + "' with new target name '" + name + "'");
					}
				}
				// 校验给定的别名是否已经存在注册的bean，如果存在抛出异常IllegalStateException
				// 当A -> B存在时，若再次出现A->C->B的时候会抛出异常
				checkForAliasCircle(name, alias);
				// 注册bean
				this.aliasMap.put(alias, name);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' registered for name '" + name + "'");
				}
			}
		}
	}
```

> 以上通过别名注册Bean的步骤：
> <br/> 1、alias与beanName相同的情况处理。Bean的name与别名相同时，不注册，并删除别名注册的Bean
> <br/> 2、alias覆盖处理。
> <br/> 3、alias循环检查。checkForAliasCircle(name, alias);
> <br/> 4、通过别名注册bean

> <br/> getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder)); 通知监听器完成bean注册，开发人员可以
> 对注册Bean事件通过监听器监听，并将处理逻辑写入监听器中

#### **alias 标签的解析**
> Spring中alias定义作用:满足不同的组件对象的调用，
> <br/> 例如:<bean id="testBean" name="testBeanA,testBeanB" class="xxx.xxx.TestBean"/>
> <br/> 组件A -> 别名testBeanA，组件B -> testBeanB
> <br/> Spring中两种别名定义方式:
> <br/> 1、通过bean标签中的属性name定义别名；
> <br/> e.g:<bean id="testBean" name="testBeanA,testBeanB" class="xxx.xxx.TestBean"/>
> <br/> 
> <br/> 2、通过<alias>标签定义别名；
> <br/> e.g:<bean id="testBean" class="xxx.xxx.TestBean"/>
> <br/> <alias name="testBean" alias="testBeanA,testBeanB"/>
> <br/>下面是alias标签解析:

```java 
    /**
	 * 处理给定的别名元素，向注册表注册别名
	 * Process the given alias element, registering the alias with the registry.
	 */
	protected void processAliasRegistration(Element ele) {
		// 获取标签的name 和 alias
		String name = ele.getAttribute(NAME_ATTRIBUTE);
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		// 判断是否为null/""/" "
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
				// 注册别名
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
			// 通知监听器别名解析完成
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}
```
                
#### **import 标签的解析**

> org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#importBeanDefinitionResource
> <import resource="text.xml" />

```java

    /**
	 * 解析并加载Bean定义资源信息
	 * Parse an "import" element and load the bean definitions
	 * from the given resource into the bean factory.
	 */
	protected void importBeanDefinitionResource(Element ele) {
		// 获取导入资源的路径:resource
		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
		if (!StringUtils.hasText(location)) {
			getReaderContext().error("Resource location must not be empty", ele);
			return;
		}

		// Resolve system properties: e.g. "${user.dir}"
		// 获取资源在当前项目的绝对路径
		location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

		Set<Resource> actualResources = new LinkedHashSet<>(4);

		// Discover whether the location is an absolute or relative URI
		// 是绝对URI 还是相对URI
		boolean absoluteLocation = false;
		try {
			// 判断资源路径是否为URL 或者 绝对路径
			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
		}
		catch (URISyntaxException ex) {
			// cannot convert to an URI, considering the location relative
			// unless it is the well-known Spring prefix "classpath*:"
		}

		// Absolute or relative?
		// 是相对路径或者绝对路径
		if (absoluteLocation) {
			try {
				// 获取导入资源的数量
				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
				}
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error(
						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
			}
		}
		else {
			// No URL -> considering resource location as relative to the current file.
			// 如果不是相对路径或者绝对路径时，将资源视为当前文件
			try {
				int importCount;
				// 得到当前资源文件的路径
				Resource relativeResource = getReaderContext().getResource().createRelative(location);
				// 资源文件存在
				if (relativeResource.exists()) {
					// 获取当前文件中导入的资源文件数量 loadBeanDefinitions加载注册bean
					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
					// 将路径添加到资源文件集合中
					actualResources.add(relativeResource);
				}
				// 资源文件不存在
				else {
					String baseLocation = getReaderContext().getResource().getURL().toString();
					importCount = getReaderContext().getReader().loadBeanDefinitions(
							StringUtils.applyRelativePath(baseLocation, location), actualResources);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
				}
			}
			catch (IOException ex) {
				getReaderContext().error("Failed to resolve current resource location", ele, ex);
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
						ele, ex);
			}
		}
		Resource[] actResArray = actualResources.toArray(new Resource[0]);
		// fireImportProcessed:通知ReaderEventListener处理完成
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}

```

#### **嵌入beans的beans标签的解析**

```xml
<beans>
    <beans>...</beans>
</beans>
```
          
{% highlight text %}
    嵌套beans标签的解析过程就是递归调用单beans标签的解析过程
{% endhighlight %} 
    













            
            
            
            
            
            
 
