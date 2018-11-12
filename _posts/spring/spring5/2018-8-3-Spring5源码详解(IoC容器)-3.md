---
layout: post
title:  "Spring5源码详解-IoC容器[三]"
date:   2018-8-3 20:00:00 +0800
tags: Spring5源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring Framework - Ioc容器'
---

{% highlight text %}
    注：基于Spring版本：5.1.x
{% endhighlight %} 

------------------------

<center><b><h3>Ioc容器源码解读</h3></b></center>

{% highlight java %}
    org.springframework.context.ApplicationContext 主要的两个实现类：
        * org.springframework.context.support.ClassPathXmlApplicationContext
        * org.springframework.context.support.FileSystemXmlApplicationContext
{% endhighlight %} 

> Ioc容器原理

![IoC容器原理图]({{ site.url }}{{site.baseurl}}/assets/img/spring/IoC.png)
{: .image-pull-right}


### 解读源码

{% highlight java %}
/** 1、使用setConfigLocations设置应用程序上下文配置路径信息 */
org.springframework.context.support.AbstractRefreshableConfigApplicationContext#setConfigLocations

/** 2、刷新配置的上线文，单例创建bean实例，可以手动调用刷新 */
org.springframework.context.support.AbstractApplicationContext#refresh()

{% endhighlight %} 

> 2、refresh()方法解读

{% highlight java %}

public void refresh() throws BeansException, IllegalStateException {
    /** startupShutdownMonitor:用于“刷新”和“销毁”的同步监视器 */
    synchronized (this.startupShutdownMonitor) {
        // 2.1 刷新容器操作之前做些准备的工作
        prepareRefresh();

        // 2.2 通知子类刷新BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 2.3 BeanFactory预处理
        prepareBeanFactory(beanFactory);

        try {
            // 2.4 bean定义被加载，没有实例化之前的处理步骤
            postProcessBeanFactory(beanFactory);

            // 2.5[重要] bean处理器：委托BeanFactoryPostProcessor给处理bean
            invokeBeanFactoryPostProcessors(beanFactory);

            // 2.6[重要] 按顺序注册bean
            registerBeanPostProcessors(beanFactory);

            // 2.7 国际化消息(I18N)初始化
            initMessageSource();

            // 2.8 Spring事件广播器初始化，可以自定义广播器
            initApplicationEventMulticaster();

            // 2.9 模板方法：由子类来实现自定义bean容器
            onRefresh();

            // 3.0 注册事件监听器：将bean容器中的事件监听器和BeanFactory中的事件监听器注册添加到事件广播器中
            registerListeners();

            // 3.1 单利模式实例化非延迟加载的bean(非@Lazy)
            finishBeanFactoryInitialization(beanFactory);

            // 3.2 调用LifecycleProcessor.onRefresh()刷新应用，加载过程结束
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            // 销毁bean
            destroyBeans();

            // Reset 'active' flag.
            // 取消激活
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            // 重置容器缓存
            resetCommonCaches();
        }
    }
}


{% endhighlight %} 

> 2.1、prepareRefresh()方法解读

{% highlight java %}

/**
*   1、设置Spring容器的启动时间，撤销关闭状态，开启活跃状态。
*   2、初始化属性源信息(Property)。
*   3、验证环境信息里一些必须存在的属性。
*/
protected void prepareRefresh() {
    // 上下文启动时的系统时间（以毫秒为单位）
    this.startupDate = System.currentTimeMillis();
    // 上下文是否关闭 false
    this.closed.set(false);
    // 上下文是否处于活动状态 true
    this.active.set(true);

    // debug:日志输出
    if (logger.isDebugEnabled()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Refreshing " + this);
        }
        else {
            logger.debug("Refreshing " + getDisplayName());
        }
    }

    // 替换ApplicationContext的根属性，默认: 不做操作
    initPropertySources();

    // 验证配置属性
    // see ConfigurablePropertyResolver#setRequiredProperties
    getEnvironment().validateRequiredProperties();

    // 初始化 ApplicationEvents
    this.earlyApplicationEvents = new LinkedHashSet<>();
}

{% endhighlight %} 

> 2.3、prepareBeanFactory(beanFactory)方法解读

{% highlight java %}

protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置BeanFactory的ClassLoader为当前的ClassLoad
    beanFactory.setBeanClassLoader(getClassLoader());
    // 设置表达式语言解析器(支持SpEL表达式 #{...}，解析bean定义中的一些表达式)
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 添加属性编辑注册器(注册属性编辑器)，属性编辑器实际上是属性的类型转换器，
    // 因为bean的属性配置都是字符串类型的 实例化的时候要将这些属性转换为实际类型 
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加BeanPostProcessor(Bean后置处理器)：ApplicationContextAwareProcessor
    // 在BEAN初始化之前，调用ApplicationContextAwareProcessor的postProcessBeforeInitialization
    // 处理所有的Aware接口，进行如下操作：
    // 如果bean实现了EnvironmentAware接口，调用bean.setEnvironment
    // 如果bean实现了EmbeddedValueResolverAware接口，调用bean.setEmbeddedValueResolver
    // 如果bean实现了ResourceLoaderAware接口，调用bean.setResourceLoader
    // 如果bean实现了ApplicationEventPublisherAware接口，调用bean.setApplicationEventPublisher
    // 如果bean实现了MessageSourceAware接口，调用bean.setMessageSource
    // 如果bean实现了ApplicationContextAware接口，调用bean.setApplicationContext
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 取消下面接口自动装配:EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、
    // ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware
    // ApplicationContextAwareProcessor把这上面的接口的实现工作做了
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 设置自动装配的特殊规则
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 添加BeanPostProcessor(后置处理器)：ApplicationListenerDetector
    // 在Bean初始化后检查是否实现了ApplicationListener接口,
    // 是则加入当前的applicationContext的applicationListeners列表
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 增加AspectJ支持
    // 检查容器中是否包含名称为"loadTimeWeaver"的bean，实际上是增加Aspectj的支持
    // AspectJ采用切面的植入方式：1、编译期植入、2、类加载期植入
    // 类加载期植入简称为LTW（Load Time Weaving）,通过特殊的类加载器来代理JVM默认的类加载器实现
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        // 添加BEAN后置处理器：LoadTimeWeaverAwareProcessor
        // 在BEAN初始化之前检查BEAN是否实现了LoadTimeWeaverAware接口，
        // 如果是，则进行加载时织入，即静态代理。
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 注入一些其它信息的bean，比如environment、systemProperties等
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}

{% endhighlight %} 


> 2.5、invokeBeanFactoryPostProcessors(beanFactory)方法解读

{% highlight java %}

protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // 2.5.1 委托BeanFactoryPostProcessor实例化bean
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    // 使用LoadTimeWeaverBean进行LWT(类植入)处理
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}

{% endhighlight %} 


> 2.5.1、PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors())方法解读

{% highlight java %}

public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    Set<String> processedBeans = new HashSet<>();

    // 2.5.1.1 判断beanFactory类型是否为BeanDefinitionRegistry
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        // 存储BeanFactoryPostProcessor类型
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        // 存储BeanDefinitionRegistryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 循环遍历：手动注册BeanFactoryPostProcessor
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            // 为BeanDefinitionRegistryPostProcessor类型
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                // 重新注册
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                // 缓存
                registryProcessors.add(registryProcessor);
            }
            // 否则直接缓存
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        // 存储BeanFactoryPostProcessor接口类
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // 第一步：获取实例化bean的名称
        // 存储实现了PriorityOrdered接口的类
        // postProcessorNames:代表Bean的容器
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            // bean类型为PriorityOrdered
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                // 写入存储器
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                // ppName的bean已经被处理
                processedBeans.add(ppName);
            }
        }
        // bean列表排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        // 将实例化的bean添加到上下文的注册表中
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // TODO 第二步, 再处理一次，将没有实例化的类重新加载一次 为什么要在做一次这一步？？？
        // 原因：存储实现了Ordered接口的类
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 最后, 调用所有其他BeanDefinitionRegistryPostProcessors，直到没有其他BeanDefinitionRegistryPostProcessors出现。
        // 存储既没有实现PriorityOrdered接口的类， 也没有实现Ordered接口的类
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // 回调postProcessBeanFactory，实例化bean
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // 通过工厂实例化bean
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // 再一次校验未初始化的bean，并缓存bean，步骤同上
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // 释放原始beanFactory中缓存的空间,新实例化的bean在容器中已经被修改，例如：替换值中的占位符...
    beanFactory.clearMetadataCache();
}

{% endhighlight %} 


> 2.6、PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);方法解读

{% highlight java %}

public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

        // 从beanFactory中获取类型为BeanPostProcessor的beanName
		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// BeanPostProcessorChecker记录信息
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
		    //先找出实现了PriorityOrdered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			//找出实现了Ordered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			// 其他接口实现加入到nonOrderedPostProcessorNames数据中
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		// 第一步、注册实现了PriorityOrdered接口的BeanPostProcessor
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		// 第二步、注册实现了Ordered接口的BeanPostProcessor
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		// 第三步, 注册所有无序的BeanPostProcessors
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		// 第四步、 注册所有MergedBeanDefinitionPostProcessor类型的BeanPostProcessors
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		// // 添加ApplicationListener监听器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}

{% endhighlight %} 

> 2.7、I18N(国际化消息)初始化 - initMessageSource();方法解读

{% highlight java %}

protected void initMessageSource() {
    // 获取BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // BeanFactory中如果存在MessageSourceBean
    if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
        this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
        // Make MessageSource aware of parent MessageSource.
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
            if (hms.getParentMessageSource() == null) {
                // Only set parent context as parent MessageSource if no parent MessageSource
                // registered already.
                hms.setParentMessageSource(getInternalParentMessageSource());
            }
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Using MessageSource [" + this.messageSource + "]");
        }
    }
    // 没有MessageSourceBean返回环境默认配置
    else {
        // Use empty MessageSource to be able to accept getMessage calls.
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
        }
    }
}

{% endhighlight %} 

> 2.8、事件广播器 - initApplicationEventMulticaster();方法解读

{% highlight java %}

protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 用户自定义事件广播器:applicationEventMulticaster
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    // 系统默认事件广播器:使用系统默认的SimpleApplicationEventMulticaster
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                    "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
    }
}

// 事件广播处理
org.springframework.context.event.SimpleApplicationEventMulticaster#multicastEvent(org.springframework.context.ApplicationEvent, org.springframework.core.ResolvableType)

public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        Executor executor = getTaskExecutor();
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            invokeListener(listener, event);
        }
    }
}

{% endhighlight %} 

> 3.0 registerListeners() - ApplicationEventMulticaster统一管理事件广播器

{% highlight java %}

protected void registerListeners() {
    // 添加容器中的事件监听器
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // 添加BeanFactory中的事件监听器
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }

    // 如果存在earlyEventsToProcess直接广播事件
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}

{% endhighlight %} 

> 3.1 finishBeanFactoryInitialization()单例模式实例化bean解析：(bean的实例化步骤会在下一章中分享)

{% highlight java %}

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化实现ConversionService接口的bean
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    // beanFactory没有设置属性解析的bean，设置默认的属性解析resolveEmbeddedValue(String)
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    // 获取实现接口LoadTimeWeaverAware的bean
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        // 实例化bean
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    // 禁止ClassLoader加载bean
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    // 禁止已经被实例化的bean做修改
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    // 确保所有的非lazy-init单例实例化，在beanFactory实例化完成时调用，
    // 如果抛出beanException，则不会实例化bean
    beanFactory.preInstantiateSingletons();
}

{% endhighlight %} 

>3.2 finishRefresh()解析：

{% highlight java %}

protected void finishRefresh() {
    // Clear context-level resource caches (such as ASM metadata from scanning).
    // 清除加载到内存的资源文件缓存
    clearResourceCaches();

    // Initialize lifecycle processor for this context.
    // 初始化容器lifecycleProcessor
    initLifecycleProcessor();

    // Propagate refresh to lifecycle processor first.
    // 调用onRefresh刷新bean，用于bean的启动或者销毁
    getLifecycleProcessor().onRefresh();

    // Publish the final event.
    // 通知所有的监听器
    publishEvent(new ContextRefreshedEvent(this));

    // Participate in LiveBeansView MBean, if active.
    // 向MBeanServer注册LiveBeansView，可以通过JMX来监控此ApplicationContext
    LiveBeansView.registerApplicationContext(this);
}

{% endhighlight %} 


<center><b><h2> 小结 </h2></b></center>

{% highlight text %}
1.IoC容器是通过单线程来初始化；
2.BeanFactory在初始化容器之前已经初始化;
3.加载Resource资源文件到缓存，实例化bean后释放资源缓存;
4.将用户定义的bean装载到Ioc容器，并转换成Ioc容器能识别的数据结构BeanDefition;
5.注册BeanDefition到Ioc容器(HashMap)缓存;
6.在这个过程并没做依赖注入(DI),在第一次调用`getBean()`时,才对非@Lazy的bean进行依赖注入;
{% endhighlight %} 