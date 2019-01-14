##  前言

在前面分析了Spring IOC的初始化过程的前半部分，今天分析一下初始化过程中一个非常重要的环节---初始化所有的 singleton beans

需要以下基础内容:

######  [Spring源码分析：Spring IOC容器初始化(一)](http://blog.ztgreat.cn/article/57)

######   [Spring-BeanFactory源码分析(一)](http://blog.ztgreat.cn/article/53)

######   [Spring-统一资源加载策略](http://blog.ztgreat.cn/article/54)

######  [Spring-BeanFactory源码分析(二)](http://blog.ztgreat.cn/article/56)

##  finishBeanFactoryInitialization

初始化 singleton beans 了，如果没有设置懒加载，那么 Spring 会在接下来初始化所有的 singleton beans。

bean的初始化过程如果要认真说的话那就很，很多地方就点到为止了，重点梳理重要的逻辑过程，其余的可以自行专研。

放出源码 `AbstractApplicationContext` -> `finishBeanFactoryInitialization`：

```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 首先，初始化名字为 conversionService 的 Bean。这里暂时不讲
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		//不管
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

        // 这是 AspectJ 相关的内容，先不管
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}
		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// 冻结 BeanDefinition，不再修改配置了
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		// 开始初始化 单例bean
		beanFactory.preInstantiateSingletons();
}
```



重点当然是 `beanFactory.preInstantiateSingletons()`，这个beanFactory是 DefaultListableBeanFactory  实例，现在我们又回到了DefaultListableBeanFactory 这个BeanFactory上了。

##  preInstantiateSingletons

放源码 `DefaultListableBeanFactory `-> `preInstantiateSingletons`:

```
public void preInstantiateSingletons() throws BeansException {
		

		// this.beanDefinitionNames 保存了所有的 beanNames
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// 触发所有的非懒加载的 singleton beans 的初始化操作
		for (String beanName : beanNames) {
		    // 合并父 Bean 中的配置，注意 <bean id="" class="" parent="" /> 中的 parent，用的不多
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			// 非抽象、非懒加载的 singletons。如果配置了 'abstract = true'，那是不需要初始化的
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			    // 处理 FactoryBean
				if (isFactoryBean(beanName)) {
				    // FactoryBean 的话，在 beanName 前面加上 ‘&’ 符号。再调用 getBean
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						// 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现，忽略算了
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
				    // 对于普通的 Bean，只要调用 getBean(beanName) 这个方法就可以进行初始化了
					getBean(beanName);
				}
			}
		}

		// 到这里说明所有的非懒加载的 singleton beans 已经完成了初始化
        // 如果我们定义的 bean 是实现了 SmartInitializingSingleton 接口的，那么在这里得到回调，忽略
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
}
```

很明显，我们需要进入到 getBean(beanName) 方法了，这个方法我们也经常用来从 BeanFactory 中获取一个 Bean，而初始化的过程也封装到了这个方法里，不过再之前呢，需要插播一下关于FactoryBean的信息，在最开始分析BeanFactory的时候，也有简单提到，这里再提一下，如果读者已经明白FactoryBean,那么可以跳过这部分。

###  FactoryBean

```
public interface FactoryBean<T> {

	@Nullable
	T getObject() throws Exception;


	@Nullable
	Class<?> getObjectType();


	default boolean isSingleton() {
		return true;
	}

}
```

FactoryBean 其实很简单，首先FactoryBean 是Bean，只是这是一个非常特殊的Bean,这种特殊的bean会生产另一种bean, 对于普通的bean,通过BeanFactory 的 getBean方法可以获取这个bean,而对于FactoryBean 来说，通过getBean 获得的是 FactoryBean 生产的bean（实际会调用FactoryBean的 `getObject`方法）,而不是FactoryBean 本身，如果想要获取FactoryBean 本身，那么可以加`前缀&`，那么spring 就明白，原来你是需要FactoryBean 。这个可能会在后面AOP的部分，展开来讲，这里就先说这么多了。



##  getBean

现在回到getBean 这上面来，代码稍微有点长。

`AbstractBeanFactory `->`getBean`:

```
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```

 ###   doGetBean

```
protected <T> T doGetBean(final String name, final Class<T> requiredType,
			final Object[] args, boolean typeCheckOnly) throws BeansException {
        // 获取一个 标准的 beanName，处理两种情况：
        //一个是前面说的 FactoryBean(前面带 ‘&’)，
        //如果指定的是别名，将别名转换为规范的Bean名称 
		final String beanName = transformedBeanName(name);
		Object bean;

		// 检查下是不是已经存在了，如果已经创建了的单例bean,会放入Map 中
		Object sharedInstance = getSingleton(beanName);
		// 但是如果 args 不为空的时候，那么不管是否该bean已经存在都会重新创建
		if (sharedInstance != null && args == null) {
			
			// 下面这个方法：如果是普通 Bean 的话，直接返回 sharedInstance，
      		// 如果是 FactoryBean 的话，返回它创建的那个实例对象，调用FactoryBean的getObject 方法
            //这里就不展开了
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// 创建过了此 beanName 的 prototype 类型的 bean，那么抛异常，
            // 往往是因为陷入了循环引用
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查一下这个 BeanDefinition 在容器中是否存在
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// 如果当前容器不存在这个 BeanDefinition，看看父容器中有没有
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
			    // typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中，标记一下。
				markBeanAsCreated(beanName);
			}
			
			/*
             * 到这里的话，要准备创建 Bean 了，
             * 对于 singleton 的 Bean 来说，容器中还没创建过此 Bean；
             * 对于 prototype 的 Bean 来说，本来就是要创建一个新的 Bean。
             */

			try {
			    //根据指定Bean名称获取其父级的Bean定义，主要解决Bean继承时子类  
                //合并父类公共属性问题
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 先初始化依赖的所有 Bean
				//检查是不是有循环依赖，这里的循环依赖和我们前面说的循环依赖又不一样
				//这里的依赖指的是 depends-on 中定义的依赖
				//depends-on用来表示一个Bean的实例化依靠另一个Bean先实例化。
				//如果在一个bean A上定义了depend-on B那么就表示：A 实例化前先实例化 B。
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 注册一下依赖关系
						registerDependentBean(dep, beanName);
						try {
						    //递归调用getBean方法，获取当前Bean的依赖Bean
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				 // 如果是 singleton scope 的，创建 singleton 的实例
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
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
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
                // 如果是 prototype scope 的，创建 prototype 的实例
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
				
				    // 如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							//回调beforePrototypeCreation方法
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
							    //回调afterPrototypeCreation方法
								afterPrototypeCreation(beanName);
							}
						});
						 //获取给定Bean的实例对象
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

		//对创建的Bean实例对象进行类型检查
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
}
```



上面，我们可以看见在创建实例时做了判断

- 如果Bean定义的单态模式(Singleton)，则容器在创建之前先从缓存中查找，以确保整个容器中只存在一个实例对象
- 如果Bean定义的是原型模式(Prototype)，则容器每次都会创建一个新的实例对象。
- 两者都不是，则根据Bean定义资源中配置的生命周期范围，选择实例化Bean的合适方法，这种在Web应用程序中 比较常用，如：request、session、application等生命周期

通过上面的代码基本上理解大概逻辑是不成问题的，接下来肯定就是分析`createBean` 方法了。

这里我们要接触一个新的类了 AbstractAutowireCapableBeanFactory，通过类名我们知道，这个是处理`@Autowired 注解`的，在spring中经常混用了 xml 和 注解 两种方式的配置方式。

###  createBean

`AbstractAutowireCapableBeanFactory`->`createBean`:

```
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// 确保 BeanDefinition 中的 Class 被加载
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// 准备方法覆写，这里又涉及到一个概念：MethodOverrides，
   		// 它来自于 bean 定义中的 <replaced-method />
   		//没怎么了解
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			//如果Bean配置了初始化前和初始化后的处理器，则试图返回一个代理对象
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
		    //创建 bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
}
```

继续看`doCreateBean` 方法吧。

###  doCreateBean

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
		    //移除BeanWrapper缓存
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
		    //创建 BeanWrapper
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//获得bean 实例
		final Object bean = instanceWrapper.getWrappedInstance();
		//获取实例化对象的类型
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		//调用PostProcessor后置处理器
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
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
		// 下面代码是为了解决循环依赖的问题
		//循环依赖问题，下篇文章来描述和举例，这里就不说了
		//所以这里如果不明白，那也没关系，放一放
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		//提前曝光bean		
		if (earlySingletonExposure) {
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
		    //实例化后，需要进行依赖注入
			populateBean(beanName, mbd, instanceWrapper);
            // 这里就是处理 bean 初始化完成后的各种回调，例如init-method 配置，BeanPostProcessor接口
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
		    //如果已经提交曝光了bean,那么就从缓存中获取bean
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
			    //根据名称获取的以注册的Bean和正在实例化的Bean是同一个
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
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
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
}
```

到这里，基本上简单的分析了 doCreateBean 方法，整个bean就已经初始化完成了，这里面有三个重点的方法（过程）

1、创建 Bean 实例(`createBeanInstance`) 方法，

2、依赖注入(`populateBean`) 方法，

3、一系列初始化或者回调(`initializeBean`)。

注意了，接下来的这三个方法要认真说那也是极其复杂的，很多地方我就点到为止了，感兴趣的读者可以自己往里看，最好就是碰到不懂的，自己写代码去调试它。



####  createBeanInstance -创建 Bean 实例 



```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

        // 校验 类的访问权限
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null)  {
		    // 采用工厂方法实例化，配置 factory-method
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		// 如果不是第一次创建，比如第二次创建 prototype bean。
        // 这种情况下，我们可以从第一次创建知道，采用无参构造函数，还是构造函数依赖注入 来完成实例化
        // 这个可以通过代码来测试，多次通过getbean(name)来获取 prototype的bean
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
			    /配置了自动装配属性，使用容器的自动装配实例化  
               //容器的自动装配是根据参数类型匹配Bean的构造方法
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
			    //使用默认的无参构造方法实例化 
				return instantiateBean(beanName, mbd);
			}
		}

		// 判断是否采用有参构造函数.
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			//使用容器的自动装配特性，调用匹配的构造方法实例化 
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		//使用默认的无参构造方法实例化
		return instantiateBean(beanName, mbd);
}
```

当然接下来看最简单的无参构成函数创建实例咯

####  instantiateBean

```
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
			     //实例化
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			// 包装一下
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			//初始化BeanWrapper
			//会设置 conversionService，注册customEditors
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
}
```

我们看到createBeanInstance方法和instantiateBean的返回值是 **BeanWrapper**，那这个BeanWrapper到底是什么？

BeanWrapper相当于一个**代理器**，Spring通过BeanWrapper完成Bean属性的填充工作。

在Bean实例被InstantiationStrategy创建出来之后，容器将Bean实例通过BeanWrapper包装起来。

BeanWrapper还有三个顶级类接口，分别是`PropertyAccessor`和`PropertyEditorRegistry`，`TypeConverter`。PropertyAccessor接口定义了各种访问Bean属性的方法，如setPropertyValue(String,Object)，setPropertyValues(PropertyValues pvs)等，而PropertyEditorRegistry是属性编辑器的注册表。

TypeConverter 支持属性值的类型转换

所以BeanWrapper实现类BeanWrapperImpl具有了多重身份：

- Bean包裹器；
- 属性访问器；
- 属性编辑器注册表。
- 属性值的类型转换



进入`beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);`看看

`SimpleInstantiationStrategy `-> `instantiate`:

```
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		
		// 如果不存在方法覆写(replaced-method 配置)，那就使用 java 反射进行实例化，否则使用 CGLIB,
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			// 利用构造方法进行实例化
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			// 存在方法覆写，利用 CGLIB 来完成实例化，需要依赖于 CGLIB 生成子类，这里就不展开了。
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
}
```

ok，到这里bean的实例化就完成了，就不继续跟进去了，完成实例化后，还有一个重要的任务就是依赖注入，这个也是一个麻烦事情。

####  populateBean - 依赖注入

`AbstractAutowireCapableBeanFactory `->`populateBean`

```
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

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
			    //InstantiationAwareBeanPostProcessor 在实例前和实例后进行回调处理
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					//在设置属性之前调用Bean的PostProcessor后置处理器 
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
        // bean 实例的所有属性
		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
			    // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			    // 通过类型装配
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}
        //检查容器是否持有用于处理单态模式Bean关闭时的后置处理器
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		//Bean实例对象没有依赖(此依赖是depends-on)，即没有继承基类 
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
				    //处理特殊的BeanPostProcessor
				    
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						//使用BeanPostProcessor处理器处理属性值  
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
			    //为要设置的属性进行依赖检查
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		if (pvs != null) {
		    // 设置 bean 实例的属性值
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
}
```

继续applyPropertyValues

####  applyPropertyValues

```
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs.isEmpty()) {
			return;
		}

		if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
		    //设置安全上下文
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			//属性值已经转换 
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
				    //为实例化对象设置属性值
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			//获取属性值对象的原始类型值
			original = mpvs.getPropertyValueList();
		}
		else {
			original = Arrays.asList(pvs.getPropertyValues());
		}
         //获取用户自定义的类型转换 
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		//创建一个Bean定义属性值解析器，将Bean定义中的属性值解析为Bean实例对象 的实际值
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// Create a deep copy, resolving any references for values.
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		boolean resolveNecessary = false;
		for (PropertyValue pv : original) {
		    //属性值已经转换
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {//属性值需要转换
			    
				String propertyName = pv.getName();
				//原始的属性值，即转换之前的属性值
				Object originalValue = pv.getValue();
				
				//转换属性值，例如将引用转换为IoC容器中实例化对象引用
				//如果属性值是其它bean,且没有创建，那么就会先去创建这个bean
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				//属性值是否可以转换
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
				    //使用用户自定义的类型转换器转换属性值
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				
				//存储转换后的属性值，避免每次属性注入时的转换工作
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
		    //标记属性值已经转换过 
			mpvs.setConverted();
		}
		try {
		    //进行属性依赖注入
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
}
```



从上面代码可以看出属性转换分为了两种情况

- 属性值类型不需要转换时，不需要解析属性值，直接准备进行依赖注入。
- 属性值需要进行类型转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

对属性值的解析是在`BeanDefinitionValueResolver`类中的`resolveValueIfNecessary`方法中进行的,这里就不再展开了，可以看到里面对list,set,map类型的解析。

###  initializeBean

属性注入完成后，就可以"初始化"bean了，这一步其实就是处理各种回调了。

```
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
		    // 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
		    // 处理 bean 中定义的 init-method，
            // 或者如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
		    // BeanPostProcessor 的 postProcessAfterInitialization 回调
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
}
```



这里有个需要注意的地方，**BeanPostProcessor** 的两个回调都发生在这边，只不过中间处理了 init-method。

到这里Bean的初始化过程总算完成了，同时IOC的初始化过程也就差不多完成了，最后还有一步 finishRefresh。



##  finishRefresh

这个简单多了,初始化一下lifecycle processor，发布容器完成初始化事件。

```
protected void finishRefresh() {
  // Clear context-level resource caches (such as ASM metadata from scanning).
  clearResourceCaches();

  // Initialize lifecycle processor for this context.
  initLifecycleProcessor();

  // Propagate refresh to lifecycle processor first.
  getLifecycleProcessor().onRefresh();

  // Publish the final event.
  publishEvent(new ContextRefreshedEvent(this));

  // Participate in LiveBeansView MBean, if active.
  LiveBeansView.registerApplicationContext(this);
}
```

至此，SpringIOC 初始化过程就完成了，整个过程还是非常复杂的，里面涉及到很多东西，只是梳理了一下整个逻辑过程，对于一些细节的地方，在需要用的时候再进行分析。

##  参考

[Spring IOC 容器源码分析](https://www.javadoop.com/post/spring-ioc)

Spring 技术内幕