## 前言

上篇文章中我们先从动态代理技术谈起，简单的分析了动态代理技术，接着对Spring 中的AOP 以ProxyFactoryBean为例，通过简单的配置，对Spring AOP的基本实现和工作流程进行了简单的梳理，通过ProxyFactoryBean 得到AopProxy对象。

对于JDK的AopProxy代理对象，使用是InvocationHandler的invoke回调入口；而对于CGLIB的AopProxy代理对象，使用的是设置好的callback回调，在这些callback 回调中，对于AOP实现，是通过DynamicAdvisedInterceptor 来完成的，而DynamicAdvisedInterceptor的回调入口是intercept方法。

今天我们的主题就是看看Spring 中Advice的实现，下面已`JdkDynamicAopProxy`为例，展开来讨论。

> 本文是在[Spring源码分析：AOP分析(一)](http://blog.ztgreat.cn/article/60)的基础上的深入

## Advice通知的实现

在上文中我们简要的分析了JdkDynamicAopProxy 中的invoke 方法，其中一行代码则是获取拦截器链：

```
// Get the interception chain for this method.
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```

这里便是我们的入口，可以看看Spring 是如何创建这个拦截器链的。

### AdvisedSupport 

这`advised`是一个AdvisedSupport 对象，这个类的继承关系如下：

![20181107203530](http://img.blog.ztgreat.cn/document/spring/20181107203530.png)

这个AdvisedSupport  类同时也是ProxyFactoryBean的基类。从AdvisedSupport的代码中可以看到`getInterceptorsAndDynamicInterceptionAdvice`的实现，在这个方法中取得了拦截器链，在取得拦截器链的时候，为提高取得拦截器链的效率，还为这个拦截器链设置了缓存。

```
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
	}
	return cached;
}
```

取得拦截器链的工作是由配置好的advisorChainFactory来完成的，从名字上可以猜到，它是一个生成通知器链的工厂。在这里，advisorChainFactory被配置成一个DefaultAdivsorChainFactory对象，在DefaultAdivsorChainFactory 中实现了interceptor链的获取过程。

### DefaultAdvisorChainFactory

```
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {

	List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
	Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
	boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
	//全局获取 AdvisorAdapterRegistry  单例
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

	for (Advisor advisor : config.getAdvisors()) {
		if (advisor instanceof PointcutAdvisor) {
			// Add it conditionally.
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				//匹配当前方法
				if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
					//适配advisor成方法拦截器
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					if (mm.isRuntime()) {
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}
	return interceptorList;
}
```

在这个获取过程中，首先设置一个List，其长度由配置的通知器的个数来决定的，这个配置就是XML中对ProxyFactoryBean做的interceptNames属性的配置。然后，DefaultAdivsorChainFactory会通过一个AdvisorAdapterRegistry来实现拦截器的注册，AdvisorAdapterRegistry对advice通知的织入功能起了很大的作用。这个在稍后将会阐述，有了AdvisorAdapterRegistry注册器，利用它来从ProxyFactoryBean 配置中得到的通知进行适配，从而获取相应的拦截器，再把它加入前面设置好的List中去，完成所谓的拦截器注册过程。在拦截器适配和注册完成以后，List中的拦截器会被JDK生成AopProxy代理对象的invoke方法或者CGLIB代理对象的intercept拦截方法取得，并启动拦截器的invoke调用，最终触发通知的切面增强。

### AdvisorAdapterRegistry

AdvisorAdapterRegistry  从 GlobalAdvisorAdapterRegistry 中获取，GlobalAdvisorAdapterRegistry的实现很简洁

```
public abstract class GlobalAdvisorAdapterRegistry {

	private static AdvisorAdapterRegistry instance = new DefaultAdvisorAdapterRegistry();


	public static AdvisorAdapterRegistry getInstance() {
		return instance;
	}

	static void reset() {
		instance = new DefaultAdvisorAdapterRegistry();
	}

}
```

从代码中可以看出，GlobalAdvisorAdapterRegistry是一个单例模式的实现，它配置了一个静态飞final变量instance,这个对象是在加载类的时候就生成的，而且GlobalAdvisorAdapterRegistry还是一个抽象类，不能被实例化，这样就保证了instance对象的唯一性。

下面看一下 DefaultAdvisorAdapterRegistry 中究竟发生了什么



```
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

    //持有一个AdvisorAdapter的list,这个List中的Adapter
    //是与Sping AOP的advice增强功能相应的
	private final List<AdvisorAdapter> adapters = new ArrayList<>(3);

	/**
	 * 这里把已有的advice实现的Adapter加进来
	 */
	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}
	//后面代码省略
}
```

在`DefaultAdvisorChainFactory`的`getInterceptorsAndDynamicInterceptionAdvice`方法中，我们看到了AdvisorAdapterRegistry   调用了getInterceptors 方法来获取方法拦截器，这个就是核心代码了。

#### getInterceptors

```

public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
	List<MethodInterceptor> interceptors = new ArrayList<>(3);
	//从Advisor通知配置中取得advice通知
	Advice advice = advisor.getAdvice();
	
	//如果是MethodInterceptor类型，不需要再适配，直接加入集合
	if (advice instanceof MethodInterceptor) {
		interceptors.add((MethodInterceptor) advice);
	}
	//对adice 进行适配
	for (AdvisorAdapter adapter : this.adapters) {
	    //适配器支持当前adive
		if (adapter.supportsAdvice(advice)) {
		    //适配并返回结果
			interceptors.add(adapter.getInterceptor(advisor));
		}
	}
	if (interceptors.isEmpty()) {
		throw new UnknownAdviceTypeException(advisor.getAdvice());
	}
	return interceptors.toArray(new MethodInterceptor[0]);
}
```

从代码中我们可以看到，最后通过adapter的getInterceptor 方法，来获取最后的拦截器，我们来看看

#### AdvisorAdapter

AdvisorAdapter的实现关系：

![20181108105933](http://img.blog.ztgreat.cn/document/spring/20181108105933.png)



从这几个适配器的名字上可以看到，它们完全和advice一一对应，在这里，它们作为适配器加入到adapters 中。

#####  MethodBeforeAdviceAdapter

已MethodBeforeAdviceAdapter为例，它的实现很简单：

```
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

	@Override
	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof MethodBeforeAdvice);
	}

	@Override
	public MethodInterceptor getInterceptor(Advisor advisor) {
		MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
		return new MethodBeforeAdviceInterceptor(advice);
	}

}
```

supportsAdvice这个方法是对advice的类型进行判断。

getInterceptor方法把advice从advisor 中取出，然后创建一个MethodBeforeAdviceInterceptor 对象，通过这个对象把advice包装起来，完成适配过程。

##### MethodBeforeAdviceInterceptor

```
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

	private final MethodBeforeAdvice advice;

	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}


	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
	    //通过advice 进行增加
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		//再继续执行 MethodInvocation的proceed 方法
		//后续拦截器调用，如果拦截器执行完毕，则会进行方法调用
		return mi.proceed();
	}

}

```

在MethodBeforeAdviceInterceptor的invoke 回调中，首先触发了advice的before回调，然后才是MethodInvocation的proceed 方法调用。



接下来我们需要回到在上篇AOP中简单分析的内容,在 JdkDynamicAopProxy-> invoken 方法中。

### 回顾 invoke 

JdkDynamicAopProxy-> invoken：

```

List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

if (chain.isEmpty()) {
 proxying.
   Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
   retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
}
else {
  
   invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
   retVal = invocation.proceed();
}
```

当我们获取到拦截器链后，如果该拦截器链不为空，那么构造ReflectiveMethodInvocation 对象，然后调用proceed 方法。



### ReflectiveMethodInvocation 

```
public Object proceed() throws Throwable {
   // We start with an index of -1 and increment early.
   
   //拦截器执行完后，再执行方法调用
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }

   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
         
   //根据拦截器的不同，执行不同的调用      
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         return proceed();
      }
   }
   else {
      // It's an interceptor, so we just invoke it: The pointcut will have
      // been evaluated statically before this object was constructed.
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

这里开始遍历拦截器链，启动对拦截器链的invoke 方法调用，根据不同的拦截器类型，执行不同的调用（Spring 会根据不同的advice类型，封装不同的拦截器）。我们例子中使用的的是MethodBeforeAdvice，因此封装的是MethodBeforeAdviceInterceptor,触发MethodBeforeAdviceInterceptor的invoke 方法调用，而在MethodBeforeAdviceInterceptor的invoke 中，会先调用advice的before方法，再继续执行 MethodInvocation的proceed 方法，继续对后续拦截器的调用，当拦截器调用完成后，再调用真正方法，这就是MethodBeforeAdvice所需要对目标对象的增强效果(在方法调用之前完成通知增强)。

通过上面的流程分析，我们知道了Advice是如果完成目标对象的增强的，在上面分析中，我们看到一些Pointcut ，MethodMatcher这些关键词，但是并没有进入深入分析，因为最开始我们只需要把握骨架就可以了，下面我们从使用的角度出发，再来进一步了解Spring AOP中的一些细节，现在有了骨架认识，再来回过头来看这些小组件就会豁然开朗了。

## PointCut

PointCut(切点)决定了Advice通知应该作用于哪个连接点，也就是说通过Pointcut来定义需要增强的方法的集合，这些集合的选取可以按照一定的规则来完成。

查看 PointCut 接口设计：

```
public interface Pointcut {
	ClassFilter getClassFilter();
	MethodMatcher getMethodMatcher();
	Pointcut TRUE = TruePointcut.INSTANCE;
}
```

 `ClassFilter getClassFilter()` ，该方法返回一个类过滤器，由于一个类可能会被多个代理类代理，于是Spring引入了责任链模式，

另一个方法则是 `MethodMatcher getMethodMatcher()` ，返回一个方法匹配器，通过某种方式来匹配方法的名称来决定是否对该方法进行增强，这就是 MethodMatcher 的作用。

默认的 Pointcut 实例，该实例对于任何方法的匹配结果都是返回 true。

###  MethodMatcher 

看一下 MethodMatcher 接口：

```
public interface MethodMatcher {
	boolean matches(Method method, @Nullable Class<?> targetClass);
	boolean isRuntime();
	boolean matches(Method method, @Nullable Class<?> targetClass, Object... args);
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```

该接口定义了静态方法匹配器和动态方法匹配器。

静态方法匹配器：它仅对方法名签名（包括方法名和入参类型及顺序）进行匹配，静态匹配仅会判别一次；

动态方法匹配器：会在运行期检查方法入参的值，而动态匹配因为每次调用方法的入参都可能不一样，所以每次都必须判断。

方法匹配器的类型由isRuntime()返回值决定，返回false表示是静态方法匹配器，返回true表示是动态方法匹配器。

## Advice 通知

Advice 定义在连接点做什么。为切面增强提供织入接口。

我们前面使用的MethodBeforeAdvice就继承至它，当然还有其它类似的方法。

## Advisor

 Advisor通知器，将 Advice 和 PointCut 结合起来,通过Advisor可以定义在哪个通知（advice）并在哪个关注点（pointcut）使用它.

```
public interface Advisor {

	Advice EMPTY_ADVICE = new Advice() {};

	Advice getAdvice();

	boolean isPerInstance();

}

```

一个重要的子接口 PointcutAdvisor

```
public interface PointcutAdvisor extends Advisor {

	Pointcut getPointcut();
	
}
```

针对前面的userService,这里我们自己写一个advisor 来把整个过程连接起来。

定义一个PointCut:

```
public class UserPointCut implements Pointcut {
    public ClassFilter getClassFilter() {
        return ClassFilter.TRUE;
    }

    public MethodMatcher getMethodMatcher() {
        return new MethodMatcher() {

            public boolean matches(Method method, Class<?> targetClass, Object[] args) {
                if (method.getName().equals("add")) {
                    return true;
                }
                return false;
            }

            public boolean matches(Method method, Class<?> targetClass) {
                if (method.getName().equals("add")) {
                    return true;
                }
                return false;
            }

            public boolean isRuntime() {
                return true;
            }
        };
    }
}
```

定义一个advice(增强逻辑)，这里采用AfterReturningAdvice

```
public class UserAfterAdvice implements AfterReturningAdvice {

    public void afterReturning(Object returnValue, Method method,
                               Object[] args, Object target) throws Throwable {
        System.out.println(
                "after " + target.getClass().getSimpleName() + "." + method.getName() + "()");
    }

}
```

定义Advisor,把advice和pointcut连接起来。

```
public class UserAdvisor implements PointcutAdvisor {

    public Advice getAdvice() {
        return new UserAfterAdvice();
    }
    public boolean isPerInstance() {
        return false;
    }
    public Pointcut getPointcut() {
        return new UserPointCut();
    }
}
```

最后XML配置:

```
<bean id="userAdvisor" class="com.study.spring.aop.PointCut.UserAdvisor"></bean>
<bean id="userService" class="com.study.spring.aop.UserServiceImpl"/>
<bean id="userServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
   <property name="targetName">
      <value>userService</value>
   </property>
   <property name="interceptorNames">
      <list>
         <value>userAdvisor</value>
      </list>
   </property>
</bean>
```

最后运行结果就不展示了，通过这样一个例子，加上前面对Advice的分析，总算是把Spring的AOP骨架弄明白了，当然很多地方，我们是忽略了的，但是这并不影响我们理解整个AOP,当有了整体认识后，再回过头来，挖掘其中的一些细节，这样才会更好的理解，而且理解也一定会更加的深刻。



## 参考

Spring 技术内幕