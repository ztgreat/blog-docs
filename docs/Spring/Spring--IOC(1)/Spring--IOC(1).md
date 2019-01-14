##  前言

在前面 先分析了最底层的IOC容器BeanFactory,接着简单分析了高级形态的容器ApplicationContext，在ApplicationContext 中我们知道一个核心方法 refresh，这里面就是IOC容器的初始化流程，在前面并没有直接去分析它，只是简单的分析了BeanDefinition的载入,解析注册，有了这些基础后，再来完整的分析IOC容器的启动流程，并在适当的时候通过例子来说明。

需要以下基础内容:

#####  [Spring-BeanFactory源码分析(一)](http://blog.ztgreat.cn/article/53)

#####  [Spring-统一资源加载策略](http://blog.ztgreat.cn/article/54)

##### [Spring-BeanFactory源码分析(二)](http://blog.ztgreat.cn/article/56)

##  refresh 源码分析

在 AbstractApplicationContext 中以及将refresh整个流程定义出来了，我们再来看refresh 源码。

```
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 同步操作
   synchronized (this.startupShutdownMonitor) {

      // 准备工作，记录下容器的启动时间、标记"已启动"状态
      prepareRefresh();

      // 这步在前面我们分析过，这步主要将配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
      // 但是此时 Bean 还没有初始化，只是配置信息都提取出来了，
      //这步中有 customizeBeanFactory 方法，具体在后面介绍
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      //这个后面会具体介绍
      prepareBeanFactory(beanFactory);

      try {
         // 如果 Bean 实现了BeanFactoryPostProcessor接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。

         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
         registerBeanPostProcessors(beanFactory);

         // 初始化当前 ApplicationContext 的 MessageSource，不是重点，忽略
         initMessageSource();

         // 初始化当前 ApplicationContext 的事件广播器，忽略
         initApplicationEventMulticaster();

         // 模板方法
         // 具体的子类可以在这里做一些特殊操作
         onRefresh();

         // 注册事件监听器，监听器需要实现 ApplicationListener 接口，忽略
         registerListeners();

         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);

         // 最后，广播事件，ApplicationContext 初始化完成
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 Bean
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);
         
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```



###  准备工作-prepareRefresh

```
protected void prepareRefresh() {
   // 记录启动时间，
   this.startupDate = System.currentTimeMillis();
   // 设置容器状态
   this.closed.set(false);
   this.active.set(true);

   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

   // Initialize any placeholder property sources in the context environment
   initPropertySources();

   // 校验 xml 配置文件
   getEnvironment().validateRequiredProperties();

   this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

这个比较简单，问题不大。

###  创建 Bean 容器，加载,注册 Bean

我们回到 refresh() 方法中的下一行 obtainFreshBeanFactory()。

注意，这个方法很重要，这里将会初始化 BeanFactory、加载 Bean、注册 Bean 等等。

在[Spring-BeanFactory源码分析(二)](http://blog.ztgreat.cn/article/56) 中我们大致分析过了这个方法，这里就简单提一下就可以了，看一下在前面文章中没有提及的方法。

`AbstractApplicationContext` -> `obtainFreshBeanFactory`:

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
   refreshBeanFactory();

   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```

`AbstractRefreshableApplicationContext`->`refreshBeanFactory`:

```
@Override
protected final void refreshBeanFactory() throws BeansException {
  
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
      // 初始化一个 DefaultListableBeanFactory
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      // 用于 BeanFactory 的序列化
      beanFactory.setSerializationId(getId());

      // 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
      customizeBeanFactory(beanFactory);

      // 加载 Bean 到 BeanFactory 中
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}
```



这里主要看一下 customizeBeanFactory 方法：

```
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
  if (this.allowBeanDefinitionOverriding != null) {
    // 是否允许 Bean 定义覆盖
  	beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
  }
  if (this.allowCircularReferences != null) {
    // 是否允许 Bean 间的循环依赖
  	beanFactory.setAllowCircularReferences(this.allowCircularReferences);
  }
}
```

BeanDefinition 的覆盖问题就是在配置文件中定义 bean 时使用了相同的 id 或 name，默认情况下，allowBeanDefinitionOverriding 属性为 null，如果在同一配置文件中重复了，会抛错，但是如果不是同一配置文件中，会发生覆盖。

循环引用也很好理解：A 依赖 B，而 B 依赖 A。或 A 依赖 B，B 依赖 C，而 C 依赖 A。

默认情况下，Spring 允许循环依赖，当然如果你在 A 的**构造方法**中依赖 B，在 B 的构造方法中依赖 A 是不行的。

之后的源码中还会出现这两个属性，这里先知道就可以了。

###  prepareBeanFactory

之前我们说过，Spring 把我们在 xml 配置的 bean 都注册以后，会"手动"注册一些特殊的 bean。

这里简单介绍下 prepareBeanFactory (factory) 方法：

```
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * @param beanFactory the BeanFactory to configure
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // BeanFactory 需要加载类，因此需要类加载器，
   // 这里设置为加载当前 ApplicationContext 类的类加载器
   beanFactory.setBeanClassLoader(getClassLoader());

   // 设置 BeanExpressionResolver
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   //关于 属性编辑器
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // 添加一个 BeanPostProcessor，这个 processor 比较简单：
   // 实现了 Aware 接口的 beans 在初始化的时候，这个 processor 负责回调，
   // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
   // 注意：它不仅仅回调 ApplicationContextAware，
   // 还会负责回调 EnvironmentAware、ResourceLoaderAware 等，看下源码就清楚了
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

   // 下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，
   // Spring 会通过其他方式来处理这些依赖。
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   /**
    * 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
    * 之前我们说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行
    * ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
    * 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
    */
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // 这个 BeanPostProcessor 也很简单，在 bean 实例化后，如果是 ApplicationListener 的子类，
   // 那么将其添加到 listener 列表中，可以理解成：注册 事件监听器
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // 这里涉及到特殊的 bean，loadTimeWeaver，忽略
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   /**
    * Spring它会帮我们默认注册一些有用的 bean，
    * 我们也可以选择覆盖
    */

   // 如果没有定义 "environment" 这个 bean，那么 Spring 会 "手动" 注册一个
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   // 如果没有定义 "systemProperties" 这个 bean，那么 Spring 会 "手动" 注册一个
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   // 如果没有定义 "systemEnvironment" 这个 bean，那么 Spring 会 "手动" 注册一个
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

不用每个地方都要看懂，我自己也只能看个大概，毕竟Spring还是很庞大的，主要了解我们平常最常用的就可以了，一些其它细节可以在大致掌握Spring 框架大概后再来进行模块化的研究。

###  postProcessBeanFactory

注册需要的BeanPostProcessor,

```
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```

在AbstractApplicationContext 中是空方法（会加载一些默认的BeanPostProcessor），扩展部分交由具体的子类来实现。

###  invokeBeanFactoryPostProcessors

下面代码中我们需要注意getBeanFactoryPostProcessors()，它是获取的手动注册的BeanFactoryPostProcessor。

```
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		//忽略
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

下面开始我们的大头，看看这个invokeBeanFactoryPostProcessors中做的都是什么，这个方法很长，其实逻辑还是很简单的，最好自己对照着源码看。

```
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();
        //判断该beanFactory 是否是 BeanDefinitionRegistry 类型
        //BeanDefinitionRegistry是可以注册Bean的
		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					//调用postProcessBeanDefinitionRegistry方法
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
			    //处理 PriorityOrdered 类型
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			//排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			//添加到要执行的集合中
			registryProcessors.addAll(currentRegistryProcessors);
			
			//调用 postProcessBeanDefinitionRegistry 方法
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
			    //处理 Ordered 类型
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
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

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		
		//获取 BeanFactoryPostProcessor
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

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
}
```

**如果 beanFactory是BeanDefinitionRegistry的实例**，这个BeanDefinitionRegistry是可以注册Bean的。然后处理我们手动注册的BeanFactoryPostProcessor，遍历每一个对象，验证是不是BeanDefinitionRegistryPostProcessor的实例。BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的子接口

```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

如果是BeanDefinitionRegistryPostProcessor的实例，那么就强制类型转换后，执行BeanDefinitionRegistryPostProcessor的生命周期方法`postProcessBeanDefinitionRegistry`，并且将这个BeanFactoryPostProcessor放入到registryPostProcessors容器中.

如果不是BeanDefinitionRegistryPostProcessor的实例，那么直接放入到regularPostProcessors容器中。

processedBeans代表了一个已经处理过的Bean的容器，防止重复执行！
获取到的BeanDefinitionRegistryPostProcessor类型的Bean中，如果是PriorityOrdered类型的，那么会放入priorityOrderedPostProcessors容器中，然后排序，执行BeanDefinitionRegistryPostProcessor的生命周期方法`postProcessBeanDefinitionRegistry`。相同的方式再处理Ordered类型的，最后剩下的再单独处理。以上三种类型的postProcessBeanDefinitionRegistry处理完后，再执行BeanFactoryPostProcessor的生命周期方法`postProcessBeanFactory`。

**如果 beanFactory不是BeanDefinitionRegistry的实例**

直接执行BeanFactoryPostProcessor的生命周期方法postProcessBeanFactory，将手动添加的BeanFactoryPostProcessor处理掉。
上面处理的是BeanDefinitionRegistryPostProcessor，下面我们再处理BeanFactoryPostProcessor

```
String[] postProcessorNames =beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

```

处理的方式同BeanDefinitionRegistryPostProcessor一致，先处理完PriorityOrdered，再处理Ordered，最后处理普通的，这里要注意的是，因为BeanDefinitionRegistryPostProcessor也是BeanFactoryPostProcessor，并且他之前已经处理过了，所以这个过程就不会在处理了。

####  举个例子



一个普通的BeanFactoryPostProcessor：

```
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    public void postProcessBeanFactory(
            ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("this is MyBeanFactoryPostProcessor -> postProcessBeanFactory......");
    }

}
```



一个普通的BeanDefinitionRegistryPostProcessor，这个BeanDefinitionRegistryPostProcessor 在 postProcessBeanDefinitionRegistry 方法中注册上面的 MyBeanFactoryPostProcessor实例。

```
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        AbstractBeanDefinition beanDefinition = new RootBeanDefinition(MyBeanFactoryPostProcessor.class);

        registry.registerBeanDefinition("myBeanFactoryPostProcessor",beanDefinition);

    }

    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        System.out.println("this is MyBeanDefinitionRegistryPostProcessor -> postProcessBeanFactory");
    }
}
```

最后在xml 中配置bean `MyBeanDefinitionRegistryPostProcessor`

```
<bean id="myBeanDefinitionRegistryPostProcessor" class="com.study.spring.bean.MyBeanDefinitionRegistryPostProcessor">
	</bean>
```

执行结果：

```
this is MyBeanDefinitionRegistryPostProcessor -> postProcessBeanFactory
//...
this is MyBeanFactoryPostProcessor -> postProcessBeanFactory......
```



这样在BeanFactory 处理 BeanFactoryPostProcessor 时，会先处理BeanDefinitionRegistryPostProcessor ，这样就会将MyBeanFactoryPostProcessor 注册到BeanFactory中，这样MyBeanFactoryPostProcessor  就会在后续处理中得到执行。

这里只是简单的用了一个例子来描述 BeanFactoryPostProcessor 的处理逻辑，具体的过程可以通过代码debug来摸索。



###  registerBeanPostProcessors

在处理完 `BeanFactoryPostProcessors` 后会注册 `BeanPostProcessors` 注意这两个区别，前者主要是面向BeanFactory的，后者主要是面向Bean的，

```
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

这里也就不展开了，和处理 BeanFactoryPostProcessors 类似，只是这里注册`BeanPostProcessors` ，并没有执行其方法，具体的执行时机是在bean实例化的时候，到这里，BeanFactory中bean并没有实例化，这个我们后面会分析到。

###  onRefresh

这是一个模板方法，具体的还有待处理的流程交由子类来实现

###  finishBeanFactoryInitialization

这个方法很重要，这里会负责初始化所有的 singleton beans，鉴于篇幅不会在本文再来分析里面的逻辑了，文章太长看着也很累，这个就留到后面后面再来分析吧。



##  总结

我们来总结一下，到目前为止，可以说 BeanFactory 已经创建完成（内部使用的是DefaultListableBeanFactory），

实现了 BeanFactoryPostProcessor 接口的 Bean 都已经初始化并且其中的 postProcessBeanFactory(factory) 方法已经得到回调执行了。而且 Spring 已经"手动"注册了一些特殊的 Bean，如 'environment'、'systemProperties' 等。

实现了BeanPostProcessor 接口的Bean都已被注册（未执行）。

接下来就是处理国际化消息，事件，监听器了，不过目前不是我们的重点，这些就暂时忽略了。

剩下的就是初始化 singleton beans 了，我们知道它们是单例的，如果没有设置懒加载，那么 Spring 会在接下来初始化所有的 singleton beans。

##  参考

[Spring IOC 容器源码分析](https://www.javadoop.com/post/spring-ioc)

Spring 技术内幕