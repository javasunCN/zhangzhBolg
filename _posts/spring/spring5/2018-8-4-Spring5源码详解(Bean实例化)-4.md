---
layout: post
title:  "Spring5源码详解—Bean实例化[四]"
date:   2018-8-4 20:00:00 +0800
tags: Spring5源码
color: rgb(77,155,37)
cover: 'https://github.com/javasunCN/zhangzhBolg/blob/master/assets/img/spring/spring.jpg?raw=true'
subtitle: 'Spring Framework - Bean实例化'
---

{% highlight text %}
    注：基于Spring版本：5.1.x
{% endhighlight %} 

------------------------

<center><b><h3>Bean实例化</h3></b></center>



{% highlight java %}
    // 初始化类 
    org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
{% endhighlight %} 

>实例化bean方法 - doGetBean解析

{% highlight java %}
    
    protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
    			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    
        // 获取bean的标准名称: 去除工厂类的前缀'&'
        final String beanName = transformedBeanName(name);
        Object bean;

        // 从singletonObjects、earlySingletonObjects、singletonFactories获取Bean对象
        Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            if (logger.isTraceEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            // 1. 获取实例化bean
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
        // 
        else {
            // bean正在被创建抛出异常，这儿使用到安全线程：ThreadLocal
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // 检查BeanFactory中是否存在bean
            BeanFactory parentBeanFactory = getParentBeanFactory();
            // BeanFactory不存在bean 并且 BeanDefinition容器中不存在bean
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                // 递归查找、实例化bean:对父级处理
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                else if (args != null) {
                    // 通过参数 -> 委托父类BeanFactory实例化bean
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else if (requiredType != null) {
                    // 没有参数 -> 委托给标准getBean实例化
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
                else {
                    return (T) parentBeanFactory.getBean(nameToLookup);
                }
            }

            if (!typeCheckOnly) {
                // 创建bean记录
                markBeanAsCreated(beanName);
            }

            try {
                // 将各种类型的BeanDefinition转换成RootBeanDefinition
                // 如果指定的Bean为子Bean的话，则合并父BEAN的属性
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                checkMergedBeanDefinition(mbd, beanName, args);

                // Guarantee initialization of beans that the current bean depends on.
                String[] dependsOn = mbd.getDependsOn();
                // bean中存在依赖bean，递归实例化依赖bean
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        // 注册依赖bean
                        registerDependentBean(dep, beanName);
                        try {
                            // 实例化依赖bean
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }

                // 实例化单例bean
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            // 2 [核心方法] 实例化bean
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    // 获取Bean
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }
                // prototype模式
                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        // 前置处理
                        beforePrototypeCreation(beanName);
                        // 实例化Bean
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        // 后置处理
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }
                // 其他模式
                else {
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // 检查实例化Bean的类型是否与实际Bean类型一致
        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
                return convertedBean;
            }
            catch (TypeMismatchException ex) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }
    	
{% endhighlight %} 

>1：getObjectForBeanInstance解析

{% highlight java %}
    protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		// 如果beanName是'&'开头
		if (BeanFactoryUtils.isFactoryDereference(name)) {
		    // NullBean类型直接返回实例化beanInstance对象
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			// 不是FactoryBean类型抛出异常：BeanIsNotAFactoryException
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
			}
		}

		// 不是FactoryBean类型 或 beanName是'&'开头，直接返回对象实例 (感觉跟上面逻辑重复)
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
		    // factoryBeanObjectCache缓存中获取实例化bean
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// 强制转换beanInstance为FactoryBean类型
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
			    // 1.1 mergedBeanDefinition获取实例化Bean,如果bean不存在时，单线程实例化bean，并添加到RootBeanDefinition
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			// factoryBeanObjectCache缓存获取bean实例,没有找到，创建实例并放入缓存
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
{% endhighlight %} 

>2.[核心方法]实例化bean - createBean

{% highlight java %}
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])

protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

    if (logger.isTraceEnabled()) {
        logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // 2.1 将BeanDefinition中的bean解析成Class引用
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        // 验证复写方法
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // 2.2 返回bean代理
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }
    
    try {
        // 2.3 如果返回的bean代理==null时，走正常的bean实例化流程
        // TODO 重要 doCreateBean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}

{% endhighlight %} 


>2.1. resolveBeanClass解析 

{% highlight java %}

protected Class<?> resolveBeanClass(final RootBeanDefinition mbd, String beanName, final Class<?>... typesToMatch)
			throws CannotLoadBeanClassException {
    try {
        // RootBeanDefinition中存在Class引用,直接返回
        if (mbd.hasBeanClass()) {
            return mbd.getBeanClass();
        }
        // JVM如果打开权限验证
        if (System.getSecurityManager() != null) {
            // 使用AccessController给创建bean赋值"特权"
            return AccessController.doPrivileged((PrivilegedExceptionAction<Class<?>>) () ->
                doResolveBeanClass(mbd, typesToMatch), getAccessControlContext());
        }
        else {
            // 2.1.1
            return doResolveBeanClass(mbd, typesToMatch);
        }
    }
    catch (PrivilegedActionException pae) {
        ClassNotFoundException ex = (ClassNotFoundException) pae.getException();
        throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), ex);
    }
    catch (ClassNotFoundException ex) {
        throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), ex);
    }
    catch (LinkageError err) {
        throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), err);
    }
}
    
{% endhighlight %} 

>2.1.1 doResolveBeanClass解析

{% highlight java %}

private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch)
			throws ClassNotFoundException {

    ClassLoader beanClassLoader = getBeanClassLoader();
    ClassLoader classLoaderToUse = beanClassLoader;
    if (!ObjectUtils.isEmpty(typesToMatch)) {
        // 当类未实例化的时候，使用临时类加载器
        ClassLoader tempClassLoader = getTempClassLoader();
        if (tempClassLoader != null) {
            classLoaderToUse = tempClassLoader;
            if (tempClassLoader instanceof DecoratingClassLoader) {
                DecoratingClassLoader dcl = (DecoratingClassLoader) tempClassLoader;
                for (Class<?> typeToMatch : typesToMatch) {
                    dcl.excludeClass(typeToMatch.getName());
                }
            }
        }
    }
    String className = mbd.getBeanClassName();
    if (className != null) {
        Object evaluated = evaluateBeanDefinitionString(className, mbd);
        if (!className.equals(evaluated)) {
            // 动态解析返回Class引用
            if (evaluated instanceof Class) {
                return (Class<?>) evaluated;
            }
            else if (evaluated instanceof String) {
                return ClassUtils.forName((String) evaluated, classLoaderToUse);
            }
            else {
                throw new IllegalStateException("Invalid class name expression result: " + evaluated);
            }
        }x
        // 临时解析器返回的Class引用直接返回，避免将bean缓存到BeanDefinition
        if (classLoaderToUse != beanClassLoader) {
            return ClassUtils.forName(className, classLoaderToUse);
        }
    }
    return mbd.resolveBeanClass(beanClassLoader);
}
{% endhighlight %} 


>2.2 doResolveBeanClass解析 - 生成bean代理是可以使用AOP来干些事情

{% highlight java %}

protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            // BeanDefinition获取Class引用
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 引用前置处理
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 应用后置处理
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        // 设置后置处理是否处理过
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}

// 前置处理
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // InstantiationAwareBeanPostProcessor实现BeanPostProcessor接口
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 用户可以实现BeanPostProcessor接口，自定义bean的前置处理
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
}

// 后置处理
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
{% endhighlight %} 

>2.3 doCreateBean(真正创建Bean实例)

{% highlight java %}
    org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// 个人观点：这是一个很巧妙的设计，为了保证bean实例化的单例特性，
		// 检查当BeanDefinition初始化为单例模式时，
		// 首先：移除factoryBean缓存中的bean
		// 其次：判断如果移除的bean，在factoryBean缓存不存在，则调用createBeanInstance实例化bean
		// 如果移除的bean存在则直接使用bean的实例
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
		    // 2.3.1 [重要]
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		// 返回对象包装的bean的实例
		final Object bean = instanceWrapper.getWrappedInstance();
		// 获取bean实例的Class引用
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
				    // 后处理合并BeanDefinition属性
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 将Class引用添加到singletonFactories、registeredSingletons
			// 提前添加到容器缓存中，解决循环依赖问题
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
		    // 2.3.2 对Bean的各种依赖和属性进行填充 比如@Autowired、@Resource、@Value注解的属性都在此进行赋值
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
		    // 获取bean的Class引用
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				    // 获取依赖项
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
		    // 设置单例Bean的销毁方式
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
    
{% endhighlight %} 

>2.3.1 `重要`createBeanInstance() - 创建Bean的实例化对象

{% highlight java %}
    protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// 将BeanDefinition中的bean解析成Class引用.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
        
        // BeanDefinition指定实例化处理器，则使用BeanDefinition中指定的
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
		    // 使用BeanDefinition中指定的实例化处理器来获取实例化bean
			return obtainFromSupplier(instanceSupplier, beanName);
		}
        
        
		if (mbd.getFactoryMethodName() != null) {
		    // 从BeanFactory中获取实例化bean
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
			    // resolvedConstructorOrFactoryMethod缓存已解析的构造函数或方法
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
			    // 基于构造函数显示注入，获取bean实例
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
			    // 使用默认的构造函数初始化bean实例
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// 没有特殊处理的情况下，只需要调用无参构造函数实例化bean对象
		return instantiateBean(beanName, mbd);
	}
    
{% endhighlight %} 


>2.3.2 populateBean 对Bean的属性、依赖赋值

{% highlight java %}
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					// 调用InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation,确认给Bean赋值
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// 按照名称@Autowire属性赋值
			if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// 按照类型@Autowire属性赋值
			if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

        // 是否包含InstantiationAwareBeanPostProcessors
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
	    // 检查依赖
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
		if (hasInstAwareBpps) {
			if (pvs == null) {
			    // 获取BeanDefinition属性值
				pvs = mbd.getPropertyValues();
			}
			// getBeanPostProcessors获取bean的Class引用
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					// 默认返回PropertyValues
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
						    // 设置PropertyDescriptors赋值给BeanWrapper，不包含忽略的依赖项
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						// 设置所依赖的属性
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
		// 需要检查依赖
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			// 依赖检查
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
		    // 解析Bean的Class引用，BeanWrapper设置属性
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
{% endhighlight %} 

<center><b><h2> 小结 </h2></b></center>

{% highlight text %}
1. Spring创建bean的两种方式：
    * singleton:单例、会尝试解决Bean的循环依赖
    * prototype:每一个请求都会实例化bean，不会解决Bean的循环依赖
2. getObjectForBeanInstance()方法实例化Bean;

{% endhighlight %} 