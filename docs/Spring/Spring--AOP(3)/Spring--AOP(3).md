## 前言

在上篇博客中，分析了Spring Aop的Advice的实现过程，其中Spring对Advice 使用了适配器模式，将Advice包装成了Interceptor，在最后，我们通过Spring提供的接口，实现了PointCut,Advice ,Advisor,至此更加的明白了三者的关系。

在前面的配置中，对于每一个需要被代理的bean，我们都是通过手动配置Advisor，很显然这是最原始的方式，当然这个从分析的角度出发也是最好分析的，今天我们稍微在那个高级一点，通过相关的配置，指定一些规则，让Spring 自动的给我们代理需要被代理的bean,这样就不再需要我们一个一个的配置了。

## 调试代码

最好的分析方式，当然还是跟着源码来看看咯，当然个人觉得在什么都不知晓的情况下，最好还是不要着眼于具体的每一行代码，一行一行的debug,这样可能会被绕晕，我个人都是反复先看几篇，了解简单流程后，对于比较混乱的地方，进行debug,然后再继续看，最后在进行一次debug,测试，验证。

说远了，回到正题，对于调试代码，我们还是延用前面最简单的代码，这里就重复贴出来了，在[Spring源码分析：AOP分析(一)](http://blog.ztgreat.cn/article/60) 中可以找到，代码也很简单（`userService`，`beforeAdvice`）

## XML配置

采用xml配置的主要原因，在于好分析。

```
<bean id="userService" class="com.study.spring.aop.UserServiceImpl"/>

<!--advice-->
<bean id="beforeAdvice" class="com.study.spring.aop.BeforeAdvice" />

<!--定义 advisor-->
<bean id="userServiceAdivsor" class="org.springframework.aop.support.NameMatchMethodPointcutAdvisor">
   <property name="advice" ref="beforeAdvice"/>
   <property name="mappedName" value="add"/>
</bean>

<!--定义DefaultAdvisorAutoProxyCreator-->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
```

### 简要说明

前4行配置，这个就是简单的配置我们需要被代理的bean,已经advice(增强)

对于这里的advisor,采用的是Spring的`NameMatchMethodPointcutAdvisor`,通过名字我们可以知道，这个是通过名称来匹配的，当然还有其他的匹配方式，比如`RegexpMethodPointcutAdvisor` 正则匹配，这个匹配规则就要丰富的很多，为了简单，当然我们还是采用NameMatchMethodPointcutAdvisor了。

在NameMatchMethodPointcutAdvisor 中，我们指定advice,以及需要被增强的方法（可以多个）。



最后我们在配置一个bean->`DefaultAdvisorAutoProxyCreator`,这个bean 会自动的根据dvisor创建代理，今天的主角就是它啦。

### 运行结果

对于userService，如果调用add 方法，那么就会执行的我们增强逻辑（即beforeAdvice）,调用其它方法，则不会增强，好了，现在该我们的主角登场了。

## DefaultAdvisorAutoProxyCreator

### 继承体系

注：下图的继承关系，删除了部分分支

![20181123111635](http://img.blog.ztgreat.cn/document/spring/20181123111635.png)



我们可以发现，DefaultAdvisorAutoProxyCreator 是一个 **BeanPostProcessor**，既然是BeanPostProcessor 那这个就好办了，前面我们已经分析IOC的流程了，在refresh中 会注册BeanPostProcessor 。

### 回顾IOC refresh

AbstractApplicationContext -> refresh:

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
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。
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
     //...省略后续代码
   }
}
```

在注册BeanPostProcessor 的时候，肯定是会实例化bean的,那么BeanPostProcessor的接口在什么时候执行呢？

```
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

### 回顾IOC 初始化bean

在 [Spring源码分析：IOC容器初始化(二)](http://blog.ztgreat.cn/article/58) 中我们分析过，为了方便，这里再贴下代码：

AbstractAutowireCapableBeanFactory ->initializeBean:

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

可想而知，Spring AOP 会在 IOC 容器创建 bean 实例（这里特指 userService）的最后对 bean 进行处理（BeanPostProcessor  调用）。其实就是在这一步进行代理增强。 

### AbstractAutoProxyCreator

我们回过头来，DefaultAdvisorAutoProxyCreator 的继承结构中，postProcessAfterInitialization() 方法在其父类 AbstractAutoProxyCreator 这一层被覆写了：

#### postProcessAfterInitialization

```
/**
 * Create a proxy with the configured interceptors if the bean is
 * identified as one to proxy by the subclass.
 * @see #getAdvicesAndAdvisorsForBean
 */
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

二话不说 继续往里看 wrapIfNecessary方法：

#### wrapIfNecessary

```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // 返回匹配当前 bean 的所有的 advisor、advice、interceptor
   // 对于本文的例子，"userService",会得到 一个 NameMatchMethodPointcutAdvisor
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      // 创建代理
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null)，这个方法将得到所有的**可用于拦截当前 bean 的**advisor、advice、interceptor。

拿到interceptor后，就可以创建代理对象了。

#### createProxy 

我们继续往下看 createProxy 方法：

```
protected Object createProxy(
      Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

   // 创建 ProxyFactory 实例
   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this);

   //我们介绍过，如果希望使用 CGLIB 来代理接口，可以配置
   // proxyTargetClass="true",这样不管有没有接口，都使用 CGLIB 来生成代理：

   if (!proxyFactory.isProxyTargetClass()) {
      if (shouldProxyTargetClass(beanClass, beanName)) {
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 1. 有接口的，调用：proxyFactory.addInterface(ifc);
         // 2. 没有接口的，调用：proxyFactory.setProxyTargetClass(true);
         // 两者在具体代理实现上策略不同（jdk proxy;cglib proxy）
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }

   // 这个方法会返回匹配了当前 bean 的 advisors 数组
   // 对于本文的例子，"userService"到这边的时候会返回 NameMatchMethodPointcutAdvisor
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   for (Advisor advisor : advisors) {
      proxyFactory.addAdvisor(advisor);
   }

   proxyFactory.setTargetSource(targetSource);
   customizeProxyFactory(proxyFactory);

   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }
   //通过 proxyFactory 获取代理
   return proxyFactory.getProxy(getProxyClassLoader());
}
```

我们看到，这个方法主要是在内部创建了一个 ProxyFactory 的实例，通过这个实例来创建代理: `getProxy(classLoader)`。

看一下 proxyFactory.getProxy(...)方法：

```
public Object getProxy(@Nullable ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```

这个方法很简单，创建AOP代理（默认工厂是 `DefaultAopProxyFactory`），然后创建代理对象，到这里就和我们在

[Spring源码分析：AOP分析(一)](http://blog.ztgreat.cn/article/60)，中分析的内容接上了，后续的创建代理过程在前面已经分析了，这里就不在继续了。

至此，关于DefaultAdvisorAutoProxyCreator的部分我们就分析完了。

##  总结

在本文，我们通过DefaultAdvisorAutoProxyCreator  自动创建bean代理，随后通过DefaultAdvisorAutoProxyCreator  的继承关系，我们知道了DefaultAdvisorAutoProxyCreator 是一个 **BeanPostProcessor**，接着我们简单的回顾了IOC中关于BeanPostProcessor 的注册以及在创建bean后执行 BeanPostProcessor  回调，最终我们知道在 BeanPostProcessor 的回调方法中，创建代理bean，具体的逻辑在DefaultAdvisorAutoProxyCreator 的父类 AbstractAutoProxyCreator中，其主要逻辑便是获取该bean的interceptor,然后创建AOPFactory来创建代理，而AOPFactory 默认是 DefaultAopProxyFactory，在创建代理过程中，会根据配置或者类的信息，选择采用cglib 代理还是jdk代理，这部分内容，在AOP 分析（一）中我们已经分析过了，至此，关于AOP的分析基本可以告一段落了，整个过程，并没有彻头彻尾的分析Spring AOP的实现。

在[Spring源码分析：AOP分析(一)](http://blog.ztgreat.cn/article/60) 中我们分析了jdk代理和cglib代理的实现原理，同时我们从ProxyFactoryBean 入手，知道了，Spring 创建代理的流程。

在 [Spring源码分析：AOP分析之Advice](http://blog.ztgreat.cn/article/61) 中，分析了Spring如何把advice(增强)应用到代理对象上的。

在本文，知道如何通过Spring 自动的为我们需要的代理的对象，创建代理，并指定过滤规则。

当然Spring AOP 中还有很多 高端的用法，但是目前这些对我们理解AOP应该足够了，如果有需要，再进行深入了解分析。