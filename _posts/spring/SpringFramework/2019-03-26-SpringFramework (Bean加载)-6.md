---
layout: post
title:  "Spring源码阅读[6]"
date:   2019-3-26 20:00:00 +0800
tags: SpringFramework源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring源码阅读[6]'
---

{% highlight text %}
    注：基于Spring版本：5.x.x
{% endhighlight %} 


<center><b><h3>SpringFramework 整体框架模块介绍</h3></b></center>

![Spring框架图]({{ site.url }}{{site.baseurl}}/assets/img/spring/spring-overview.png)
{: .image-pull-right}

------------------------

<center><b><h3>Bean加载</h3></b></center>

### **Bean的加载方式** 
{% highlight text %}
首先，我们来熟悉下Spring框架加载Bean的方式:
    [转载自：http://www.cnblogs.com/lizo/p/6759080.html]
    一、定义Bean的方式
        常见的定义Bean的方式：
            1、通过xml的方式
                <bean id="myBean" class="com.xx.xx.MyBean"/>
            2、通过注解的方式，在Class上使用@Component等注解
                @Component
                public class xxxServicer{
                  ....
                }
            3、通过在@Configuration类下的@Bean的方式
                @Configuration
                public class xxxConfiguration{
                  @Bean
                  public myBean myBean(){
                      return new myBean();
                  }
                }
        虽然这三种定义Bean的方式不一样，对应的处理细节也不一样，但是从大的逻辑上来看，都是一样。
        主要的流程如下： 最关键的就是问题就是这么去找到定义Bean的方式，然后生成BeanDefinition后注册到Spring上下文中，由Spring自动创建Bean的实例。
    
    二、BeanDefinition
        BeanDefinition是一个接口，用来描述一个Bean实例，例如是SINGLETON还是PROTOTYPE，属性的值是什么，构造函数的参数是什么等。
        简单来说，通过一个BeanDefinition我们就可以完成一个Bean实例化。 BeanDefinition及其主要的子类：
        
        RootBeanDefinition和ChildBeanDefinition： 
            这2个BeanDefinition是相对的关系，自Spring 2.5 出来以后，已经被GenericBeanDefinition代替。因为这样强迫我们在编写代码的时候就必须知道他们之间的关系。
        GenericBeanDefinition: 
            相比于RootBeanDefinition和ChildBeanDefinition在定义的时候就必须硬编码，GenericBeanDefinition的优点可以动态的为GenericBeanDefinition设置parent。
        AnnotatedBeanDefinition：
            看名字就是知道是用来读取通过注解定义Bean。
          
    三、通过xml文件定义Bean
        通过xml定义Bean是最早的Spring定义Bean的方式。因此，怎么把xml标签解析为BeanDefinition()， 
        入口是在org.springframework.beans.factory.xml.XmlBeanDefinitionReader这个类，
        但是实际干活的是在org.springframework.beans.factory.xml.BeanDefinitionParserDelegate。
        代码很多，但实际逻辑很简单，就是解析Spring定义的<bean> <property> 等标签 。 
            加载Bean逻辑：
                1、封装资源文件。当进入 XmlBeanDefinitionReader 后首先对参数 Resource 使用 EncodedResource类进行封装。
                2、获取输入流 。 从 Resource 获取对应的 InputStrearm 并构造 InputSource。
                3、通过构造的 InputSource 实例和 Resource 实例继续调用函数 doLoadBeanDefinitions。
    四、通过@Component等Spring支持的注解加载Bean
        如果要使用@Component等注解定义Bean，一个前提条件是:有<context:component-scan/>或者@ComponentScan注解。但这2个方式还是有一点点区别：
        4.1 <context:component-scan/>
        由于<context:component-scan/>是一个xml标签，因此是在解析xml，
        生成的类org.springframework.context.annotation.ComponentScanBeanDefinitionParser，关键代码:
        
        ``` java
            @Override
            public BeanDefinition parse(Element element, ParserContext parserContext) {
                    //获取base-package标签
                String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
                basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
                String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
                        ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
            
                // 实际处理类是ClassPathBeanDefinitionScanner 
                ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
                //扫描basePackage下所有的类，如果有@Component等标签就是注册到Spring中
                Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
                registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
                return null;
            }
        ```
        
        4.2 @ComponentScan
        注解对应生成的类是org.springframework.context.annotation.ComponentScanAnnotationParser 
        其实最后实际干活的还是ClassPathBeanDefinitionScanner这个。
        ComponentScanAnnotationParser类的生成是伴随着@Configuration这个注解处理过程中
        （意思说@ComponentScan必须和@Configuration一起使用）。
        而处理@Configuration其实是org.springframework.context.annotation.ConfigurationClassPostProcessor。是不是感觉有点绕。
        其实简单来说，在处理@Configuration的时候发现有@ComponentScan注解，就会生成ComponentScanAnnotationParser去扫描@Component注解
        
        4.3 ClassPathBeanDefinitionScanner
        上面说到了，无论注解还是标签的方式，最后都会交给ClassPathBeanDefinitionScanner这个类来处理，
        这个类做的就是
            1.扫描basePackage下所有class，如果有@Component等注解，读取@Component相关属性，生成ScannedGenericBeanDefinition，注册到Spring中
    五、通过@Bean方式
        前面说了@ComponentScan是在@Configuration处理过程中的一环，既然@Bean注解也是必须和@Configuration一起使用，
        那么说明@Bean的处理也是在@Configuration中，其实最后是交给ConfigurationClassBeanDefinitionReader这个类来处理的，关键代码：
        
        ```java
            private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
                    TrackedConditionEvaluator trackedConditionEvaluator) {
            
                //如果自己是通过@Import注解定义的，那么需要把自己注册到Spring中
                if (configClass.isImported()) {
                    registerBeanDefinitionForImportedConfigurationClass(configClass);
                }
                //这里就是处理方法上的@Bean
                for (BeanMethod beanMethod : configClass.getBeanMethods()) {
                    loadBeanDefinitionsForBeanMethod(beanMethod);
                }
                //处理@ImportResource，里面解析xml就是上面说到的解析xml的XmlBeanDefinitionReader
                loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
                loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
            }
        
    
        ```
        
    六、把BeanDefinition实例化
        前面分别说了怎么把不同定义Bean的方式转换为BeanDefinition加入到Spring中去（确切来说是保持在BeanFactory的BeanDefinitionMap中），
        实例化这些BeanDefinition是在ApplicationContext最后阶段，关键代码在DefaultListableBeanFactory中
        
        ```java
              @Override
              public void preInstantiateSingletons() throws BeansException {
                 for (String beanName : beanNames) {
                    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
                    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                        if (isFactoryBean(beanName)) {
                            final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                            boolean isEagerInit;
                            if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                                isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                                    @Override
                                    public Boolean run() {
                                        return ((SmartFactoryBean<?>) factory).isEagerInit();
                                    }
                                }, getAccessControlContext());
                            }
                            else {
                                isEagerInit = (factory instanceof SmartFactoryBean &&
                                            ((SmartFactoryBean<?>) factory).isEagerInit());
                            }
                            if (isEagerInit) {
                                getBean(beanName);
                            }
                        }
                        else {
                            getBean(beanName);
                        }
                    }
                }
            }
        
        ```
        
            通过getBean实例化BeanFactory，代码是在AbstractAutowireCapableBeanFactory中
            
        ```java
            protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
                //处理xxAware接口
                if (System.getSecurityManager() != null) {
                    AccessController.doPrivileged(new PrivilegedAction<Object>() {
                        @Override
                        public Object run() {
                            invokeAwareMethods(beanName, bean);
                            return null;
                        }
                    }, getAccessControlContext());
                }
                else {
                    invokeAwareMethods(beanName, bean);
                }
                Object wrappedBean = bean;
                if (mbd == null || !mbd.isSynthetic()) {
                    // 调用BeanPostProcessors#postProcessBeforeInitialization
                    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
                }
                try {
                   //初始化，先判断是否是InitializingBean，
                    invokeInitMethods(beanName, wrappedBean, mbd);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(
                            (mbd != null ? mbd.getResourceDescription() : null),
                            beanName, "Invocation of init method failed", ex);
                }
                if (mbd == null || !mbd.isSynthetic()) {
                    // 调用BeanPostProcessors#postProcessAfterInitialization
                    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
                }
                return wrappedBean;
            }
        ```
        
           总结： 综上分析，Spring加载Bean其实大的思想都是一样的，先读取相关信息生成BeanDefinition，然后通过BeanDefinition初始化Bean。
                如果知道了上面了套路以后，就可以清楚怎么自定义Xml标签或者自定义注解向Spring中注入Bean。
{% endhighlight %}


#### **Bean的依赖注入方式** 
{% highlight text %}
    Spring中依赖注入有三种注入方式：
    
        一、构造器注入；
        
        二、设值注入（setter方式注入）；
        
        三、Feild方式注入（注解方式注入）。
{% endhighlight %}


#### **Bean加载过程分析(XML方式)**
> 1、通过XmlBeanFactory的构造方法，传入Resource资源
```java
    public XmlBeanFactory(Resource resource) throws BeansException {
        // 构造函数再次调用构造方法
        this(resource, null);
    }
```
> 2、调用XmlBeanFactory多参构造函数，完成Bean的加载
```java
    public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
        // 调用父级构造，忽略给定接口的自动装配功能
        super(parentBeanFactory);
        this.reader.loadBeanDefinitions(resource);
    }
```
>   注：super(parentBeanFactory); 实现类 org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#AbstractAutowireCapableBeanFactory()
```java
    public AbstractAutowireCapableBeanFactory(@Nullable BeanFactory parentBeanFactory) {
        this();
        setParentBeanFactory(parentBeanFactory);
    }
    // this(); ->
    public AbstractAutowireCapableBeanFactory() {
        super();
        ignoreDependencyInterface(BeanNameAware.class);
        ignoreDependencyInterface(BeanFactoryAware.class);
        ignoreDependencyInterface(BeanClassLoaderAware.class);
    }
```
>   其中方法:ignoreDependencyInterface(**.class)功能：忽略给定接口的自动装配功能
>   主要作用：忽略实现了BeanNameAware、BeanFactoryAware、BeanClassLoaderAware的Bean的依赖注入，
>           通过其他方式来实现spring上下文的依赖注入

> 3、[核心方法] org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource)
```$xslt
Resource读取流程(准备工作)：
    1、封装资源文件。调用XmlBeanDefinitionReader的方法loadBeanDefinitions,使用EncodedResource对资源文件进行封装
    2、获取输入流。从封装EncodedResource获取输入流 InputStream,封装InputStream为InputSource
    3、解析XML文件。调用方法 doLoadBeanDefinitions(InputSource inputSource, Resource resource)，如下：
```

```java

    // 1
    @Override
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        // 将Resource进行编码处理返回编码后的资源实体EncodedResource
        return loadBeanDefinitions(new EncodedResource(resource));
    }
```

> 使用EncodedResource对资源文件编码进行处理，核心方法 getReader()

```java
    public Reader getReader() throws IOException {
		if (this.charset != null) {
		    // java.io.InputStreamReader按照编码读取
			return new InputStreamReader(this.resource.getInputStream(), this.charset);
		}
		else if (this.encoding != null) {
		    // java.io.InputStreamReader按照编码读取
			return new InputStreamReader(this.resource.getInputStream(), this.encoding);
		}
		else {
			return new InputStreamReader(this.resource.getInputStream());
		}
	}
```

> 获取到读入流的使用，再调用内部方法 loadBeanDefinitions(EncodedResource) 如下:

```java    
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (logger.isInfoEnabled()) {
            logger.info("Loading XML bean definitions from " + encodedResource);
        }
        // 获取前线程Resource集合(通过属性来记录加载的资源)
        Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        // 当前集合的Resource集合是null，将encodedResource添加到当前集合
        if (currentResources == null) {
            currentResources = new HashSet<>(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }
        // 当前线程Resource集合存在encodedResource，不需要再加载
        if (!currentResources.add(encodedResource)) {
            throw new BeanDefinitionStoreException(
                    "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        }
        try {
            // 获取encodedResource的输入流
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                // 将输入流（inputStream）封装成InputSource
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                // 实现Bean加载
                return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            }
            finally {
                // 关闭输入流
                inputStream.close();
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "IOException parsing XML document from " + encodedResource.getResource(), ex);
        }
        finally {
            // Bean初始化完成后，移除资源配置
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                // 移除资源属性(防止重复加载)
                this.resourcesCurrentlyBeingLoaded.remove();
            }
        }
    }
```

> 整理准备阶段的逻辑：传入资源文件(Resource)，对于资源文件有编码的要求，使用 EncodedResource 对资源文件进行编码，从编码后获取资源文件
> 输入流，将输入流传入 资源文件处理核心方法 doLoadBeanDefinitions(inputSource, encodedResource.getResource()),下面来分析下:
```java
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			Document doc = doLoadDocument(inputSource, resource);
			return registerBeanDefinitions(doc, resource);
		}
		......
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

> 调用内部方法 doLoadDocument(inputSource, resource) 获取解析后的 Document，调用 内部方法 registerBeanDefinitions(doc, resource)
> 注册Bean到容器，分析如下:
{% highlight java %}
 
    // 获取Document
    protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
	// 注册Bean
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        int countBefore = getRegistry().getBeanDefinitionCount();
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        return getRegistry().getBeanDefinitionCount() - countBefore;
    }
	
{% endhighlight %}

> 上面的2个方法主要做了下面2件事:
>   1、通过对XML验证，解析XML获取Document
>   2、解析XML返回的Document，并注册Bean

> XML解析复杂，下面细细道来
> 第一步:校验XML(XML的验证模式)
>   XML的验证模式通常有两种：DTD和XSD

>       DTD验证模式：DTD ( Document Type Definition )即文挡类型定义，是一种 XML 约束模式语言，是 XML 文件的验证机制，
>       属于 XML 文件组成的一部分。
{% highlight xml %}
    <?xml versioπ=”1.0” encod1ng=”UTF 8”?>
    <!DOCTYPE beans PUBLIC "-//Sp口 ng//DTD BEAN 2 0//EN" dtd/ Spring-beans-x.dtd">
    <beans>...</beans>
{% endhighlight %}


>       XSD验证模式：XML Schema 语言就是 XSD ( XML Schemas Definition )。 XML Schema描述了 XML 文梢 的结构。 可以用一个指定的XML Schema来验证某个XML文档， 
>       以检查该XML文档是否符 合其要求。 文档设计者可以通过 XMLSchema指定 XML文档所允许的结构和内容，并可据此 检查 XML 文档是否是有效的 。
>       XML Schema 本身是 XML 文档 ， 它符合 XML 语法结构 。 可以 用通用的 XML 解析器解析它 。
>       在使用 Schema 文档对 实例文档进行检验，除了要声明名称空间外 
>       (xmlns="http://www.springframework.org/schema/beans)，还必须指定该名称空间所对应的 XML Schema文 挡的存储位置。 
>       通过 schemaLocation属性来指定名称空间所对应的 XML Schema文档的存储位置 ， 它包含两个部分， 
>            一部分是名称空间的 URI，
>             另一部分就是该名称空间所标识的 XML Schema 文件位置或 URL地址( 
>              xsi:schemaLocation="http://www.springframework.org/schema/beans
>                         http://www.springframework.org/schema/beans/spring-beans.xsd"
>              )。
>           SAX:SAX（simple API for XML）是一种XML解析的替代方法。相比于DOM，SAX是一种速度更快，更有效的方法。它逐行扫描文档，一边扫描一边解析。

{% highlight xml %}

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
               http://www.springframework.org/schema/beans/spring-beans.xsd">
        ......
    </beans>
    
{% endhighlight %}

> Spring如何解析XML

{% highlight text %}
    1、DefaultDocumentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,getValidationModeForResource(resource), isNamespaceAware());
    参数1：inputSource:资源文件输入流
    参数2：getEntityResolver():获取解析器 EntityResolver
    参数3：errorHandler：SAX 错误处理程序的基本接口
    参数4：getValidationModeForResource(resource): 获取资源的验证模式
    参数5：isNamespaceAware()：返回XML解析器是否应该支持XML命名空间。
{% endhighlight %}

> 参数2：getEntityResolver() -> 何为 EntityResolver?

{% highlight text %}
    何为 EntityResolver?官网这样解释. 如果 SAX 应用程序需要实现自定义处理外部实体，则必须实现此接口并使用 setEntityResolver 方法向 SAX 驱动器注册一个实例 。 
    也就是说，对于解析一个 XML, SAX 首先读取该 XML 文档上的声明，根据声明去寻找相应的 DTD定义，以便对文档进行一个验证。 
    默认的寻找规则， 即通过网络(实现上就是声明的 DTD 的 URI 地址)来下载相应的 DTD 声明，并进行认证。 
    下载的过程是一个漫长的过程，而且当网络中断或不可用时，这里会报错，就是因为相应的 DTD 声明没有被找到的原因 。
    EntityResolver的作用是项目本身就可以提供一个如何寻找 DTD 声明的方法，即由程序来 实现寻找 DTD 声明的过程，比如我们将 DTD 文件放到项目中某处 ，
    在实现时直接将此文档读取并返回给 SAX 即可。 这样就避免了通过网络来寻找相应的声明。
    
    首先看 entityResolver 的接口方法声明 :
        InputSource resolveEntity (String publi cid, Stri口g systernid)
    这里，它接收两个参数 publicld 和 systemId，并返回一个 inputSource对象。 这里我们以特定配置文件来进行讲解 。
    1. 如果我们在解析验证模式为 XSD 的配置文件，代码如下: 
        <beans xmlns="http://www.springframework.org/schema/beans"
    	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	   xsi:schemaLocation="http://www.springframework.org/schema/beans
               http://www.springframework.org/schema/beans/spring-beans.xsd">
    读取到以下两个参数。 
    publicld: null
    systemId : http://www.springframework.org/schema/beans/spring-beans.xsd
    2. 如果我们在解析验证模式为 DTD 的配置文件，代码如下:
    
    <!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN"
    			"http://www.springframework.org/dtd/spring-beans-2.0.dtd">
    读取到以下两个参数。
    publicld: -//SPRING//DTD BEAN 2.0//EN
    systemId : http://www.springframework.org/dtd/spring-beans-2.0.dtd
        
    之前已经提到过，验证文件默认的加载方式是通过 URL 进行网络下载获取，这样会造成延迟， 用户体验也不好， 
    一般的做法都是将验证文件放置在自己的工程里，那么怎么做才能将 这个 URL 转换为自己工程里对应的地址文件呢?
    我们以加载 DTD 文件为例来看看 Spring 中是 如何实现的 。 
    根据之前 Spring 中通过 getEntityResolver()方法对 EntityResolver 的获取，我们知道， Spring中使用 DelegatingEntityResolver类为 EntityResolver 的实现类， 
    resolveEntity 实现方法如下:
{% endhighlight %}

> DelegatingEntityResolver.resolveEntity(publicId, systemId)

```java
    public InputSource resolveEntity(@Nullable String publicId, @Nullable String systemId)
    			throws SAXException, IOException {
    
    		if (systemId != null) {
    			// 解析DTA文件
    			if (systemId.endsWith(DTD_SUFFIX)) {
    				return this.dtdResolver.resolveEntity(publicId, systemId);
    			}
    			// 解析schema文件 调用/META- INF/Spring . schemas 解析
    			else if (systemId.endsWith(XSD_SUFFIX)) {
    				return this.schemaResolver.resolveEntity(publicId, systemId);
    			}
    		}
    
    		// Fall back to the parser's default behavior.
    		return null;
    	}
```

> BeansDtdResolver.resolveEntity(publicId, systemId)

```java 
    public InputSource resolveEntity(@Nullable String publicId, @Nullable String systemId) throws IOException {
		if (logger.isTraceEnabled()) {
			logger.trace("Trying to resolve XML entity with public ID [" + publicId +
					"] and system ID [" + systemId + "]");
		}

		if (systemId != null && systemId.endsWith(DTD_EXTENSION)) {
			int lastPathSeparator = systemId.lastIndexOf('/');
			int dtdNameStart = systemId.indexOf(DTD_NAME, lastPathSeparator);
			// 如果是spring-beans解析路径
			if (dtdNameStart != -1) {
				// 获取spring-beans.dtd文件
				String dtdFile = DTD_NAME + DTD_EXTENSION;
				if (logger.isTraceEnabled()) {
					logger.trace("Trying to locate [" + dtdFile + "] in Spring jar on classpath");
				}
				try {
					// 通过相对路径获取spring-beans.dtd的文件流
					Resource resource = new ClassPathResource(dtdFile, getClass());
					InputSource source = new InputSource(resource.getInputStream());
					source.setPublicId(publicId);
					source.setSystemId(systemId);
					if (logger.isDebugEnabled()) {
						logger.debug("Found beans DTD [" + systemId + "] in classpath: " + dtdFile);
					}
					return source;
				}
				catch (FileNotFoundException ex) {
					if (logger.isDebugEnabled()) {
						logger.debug("Could not resolve beans DTD [" + systemId + "]: not found in classpath", ex);
					}
				}
			}
		}

		// Fall back to the parser's default behavior.
		return null;
	}
```


> 参数4：getValidationModeForResource(resource): 获取资源的验证模式

```java
    protected int getValidationModeForResource(Resource resource) {
        // 获取自定义的XML验证模式
        int validationModeToUse = getValidationMode();
        // 使用手动指定的验证模式
        if (validationModeToUse != VALIDATION_AUTO) {
            return validationModeToUse;
        }
        // 自动判断验证模式
        int detectedMode = detectValidationMode(resource);
        if (detectedMode != VALIDATION_AUTO) {
            return detectedMode;
        }
        // 默认 文件中 不 存在"DOCTYPE"标识采用XSD验证
        return VALIDATION_XSD;
    }
```

> 自动判断验证模式：调用org.springframework.util.xml.XmlValidationModeDetector#detectValidationMode

```java
    public int detectValidationMode(InputStream inputStream) throws IOException {
        // 查找DOCTYPE标记
        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
        try {
            boolean isDtdValidated = false;
            String content;
            while ((content = reader.readLine()) != null) {
                content = consumeCommentTokens(content);
                // 当前行在注释内或者没有内容，跳出当前循环
                if (this.inComment || !StringUtils.hasText(content)) {
                    continue;
                }
                // 如果存在DOCTYPE标记，使用DTD解析
                if (hasDoctype(content)) {
                    isDtdValidated = true;
                    break;
                }
                // 解析XML是否有开始标记
                if (hasOpeningTag(content)) {
                    break;
                }
            }
            return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
        }
        catch (CharConversionException ex) {
            return VALIDATION_AUTO;
        }
        finally {
            reader.close();
        }
    }
```

> 获取Document DefaultDocumentLoader.loadDocument方法,采用SAX解析XML

```java
    public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
		//创建DocumentBuilder工程
		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isDebugEnabled()) {
			logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		// 创建一个JAXP DocumentBuilder
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		// DocumentBuilder解析资源文件
		return builder.parse(inputSource);
	}
```

> 上面是对XML文件的读取准备阶段，下面解析并注册 BeanDefinitions (org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions)

```java
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		// 实例化BeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 记录统计前BeanDefinition的加载个数
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 加载及注册Bean
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 记录本机加载Bean的个数
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

> createBeanDefinitionDocumentReader() -> 返回：DefaultBeanDefinitionDocumentReader

```java
    protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
		return BeanUtils.instantiateClass(this.documentReaderClass);
	}
```

> org.springframework.beans.factory.xml.BeanDefinitionDocumentReader#registerBeanDefinitions

```java
    public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		// 获取根节点<beans></beans>
		Element root = doc.getDocumentElement();
		doRegisterBeanDefinitions(root);
	}
```

> [核心方法]调用内部方法doRegisterBeanDefinitions(root),底层解析XML，解析根标签(<beans></beans>)获取需要注册的bean定义

```java
    protected void doRegisterBeanDefinitions(Element root) {
		// 委托,解析XML结构
		BeanDefinitionParserDelegate parent = this.delegate;
		// 创建委托解析器
		this.delegate = createDelegate(getReaderContext(), root, parent);
		// 判断是否为Bean的默认命名空间: http://www.springframework.org/schema/beans
		if (this.delegate.isDefaultNamespace(root)) {
			// 获取当前Bean作用的环境
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				//通过逗号分隔，可以定义多个环境值
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// 判断当前环境是否包含定义的环境属性
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		// Bean解析前置处理，方法为空，使用模板方法模式，继承此类的对象可以重写此方法
		preProcessXml(root);
		// 解析Bean
		parseBeanDefinitions(root, this.delegate);
		// Bean解析后置处理，方法为空，使用模板方法模式，继承此类的对象可以重写此方法
		postProcessXml(root);

		// 重置解析代理
		this.delegate = parent;
	}
```

> [核心方法]org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions(Element, BeanDefinitionParserDelegate)
> 此方法解析解析根元素(<beans>):"import", "alias", "bean",逻辑如下:

```java
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
						// 解析根标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			// 解析根标签
			delegate.parseCustomElement(root);
		}
	}
```

> [核心方法]org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate)
> 解析bean命名空间定义的标签:"import", "alias", "bean",如果标签为<beans>递归解析

```java
    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			// 加载导入(import)标签资源文件
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			// 处理指定别名元素，注册别名到BeanFactory
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			// 解析并注册Bean
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// 递归调用解析
			doRegisterBeanDefinitions(ele);
		}
	}
```

> 下面针对不同的标签解析方法，进行单独的分析:
> 
> 1、"import":导入资源文件标签,例如

{% highlight text %}
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        　 xsi:schemaLocation="http://www.springframework.org/schema/beans 
    　　　　http://www.springframework.org/schema/beans/spring-beans.xsd">
    
        <!-- Spring中引入其他配置文件 -->
        <import resource="classpath*:/spring/job-timer.xml" />
        
    </beans>
    
    在XML有两种声明方式:
        方式一(默认): <bean id=”myBean” class=”xxx.xxx.MyBean”/>
        方式二(注解扫描): <tx:annotation-driven/>
        
    这两种方式的读取及解析差别是非常大的，如果采用 Spring 默认的配置 ， 
    Spring 当然知道该怎么做，但是如果是自定义的，那么就需要用户实现一些接口及配置。 
    对于根节点或者子节点如果是默认命名空间的话 则采用 parseDefaultElement 方法进行解析，
    否则使用 delegate.parseCustomElement 方法对自定义命名空间进行解析。 而判断是否默认命名空间还是
    自定义命名空间的办法其实是使用 node.getNamespaceURI()获取命名空间，
    并与 Spring 固定的命名空间 http://www.springframework.org/scherna/beans 进行比对。 
    如果一致则认为是默认，否则就认为是自定义。 而对于默认标签解析与自定义标签解析我们将会在下一章中进行讨论。
{% endhighlight %}

```java
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
		// fireImportProcessed:通知ReaderEventListener 导入处理完成
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}
```

> 2."alias":注册别名到BeanFactory

```java
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

> 3、"bean":解析并加载Bean

```java
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		// 解析<bean>标签中的每个属性，例如 class、name、id、alias 之类的属性。
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			// 解析自定义属性
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				// [核心]委托BeanDefinitionReaderUtils注册Bean
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

### **总结**
{% highlight text %}
    1、Bean的解析注册过程：传入Resource资源信息并进行编码处理EncodedResource, 将处理后的资源文件输入流传入loadBeanDefinitions(EncodedResource),
        通过输入流获取资源文件Document对象，BeanDefinitionDocumentReader注册Bean到BeanFactory
    2、XML的验证模式通常有两种：DTD和XSD
    3、Spring中依赖注入有三种注入方式：
          一、构造器注入；
          二、设值注入（setter方式注入）；
          三、Feild方式注入（注解方式注入）。

{% endhighlight %}       
            
            
            
            
            
            
            
            
            
 
