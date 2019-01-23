## 前言

Spring 的两大核心，一是IOC，我们之前已经学习过，而另一个则是大名鼎鼎的 AOP，AOP的具体概念我就不介绍了，我们今天重点是要从源码层面去看看 spring 的 AOP 是如何实现的。

在Spring AOP实现中，使用的核心技术是动态代理，生成代理类有两种策略：jdk动态代理和cglib动态代理。

下面简要的谈一谈这两种代理

## JDK动态代理

jdk 代理是基于接口实现的，最好的学习方式当然是看源码，因此这里我们可以将生成的代理类导出，再通过反编译查看代码，看看里面是怎么回事。

定义一个`UserService`接口:

```
public interface UserService {

    void add();

}
```

实现类`UserServiceImpl`：

```
public class UserServiceImpl implements UserService {

    public void add() {
        System.out.println("--------------------add----------------------");
    }
}
```

编写自己的`InvocationHandler`:

```
public class MyInvocationHandler implements InvocationHandler {

    private Object target;
    public MyInvocationHandler(Object target) {
        super();
        this.target = target;
    }
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("-----------------begin "+method.getName()+"-----------------");
        Object result = method.invoke(target, args);
        System.out.println("-----------------end "+method.getName()+"-----------------");
        return result;
    }
    public Object getProxy(){
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), target.getClass().getInterfaces(), this);
    }

}
```

将代理类导出：

```
public class JdkProxyTest {

    public static void main(String[] args) {

        //生成的代理类保存到磁盘
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        UserService service = new UserServiceImpl();
        MyInvocationHandler handler = new MyInvocationHandler(service);
        UserService proxy = (UserService) handler.getProxy();
        proxy.add();
    }

}
```

现在JDK动态代理已经完成了，现在我们来看看生成的代理类反编译结果：

```
public final class $Proxy0 extends Proxy implements UserService {
    //省略部分代码
    private static Method m3;
    public $Proxy0(InvocationHandler h) throws  {
        super(h);
    }
    //代理类的add 方法
    public final void add() throws  {
        try {
            //调用 InvocationHandler 的invoke方法
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
    
            m3 = Class.forName("com.study.spring.proxy.jdk.UserService").getMethod("add", new Class[0]);
            //省略部分代码
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

通过代码很容易就明白了，JDK动态代理是**使用接口**生成新的实现类，实现类里面则委托给InvocationHandler，InvocationHandler里面则调用被代理的类方法。



## CGLIB 动态代理

同JDK代理一样，通过代码来理解

编写方法拦截器：

```
public class CglibProxy implements MethodInterceptor {
    private Enhancer enhancer = new Enhancer();
    public Object getProxy(Class clazz) {
        enhancer.setSuperclass(clazz);
        enhancer.setCallback( this);
        return enhancer.create();
    }

    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("-----------------begin "+method.getName()+"-----------------");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("-----------------end "+method.getName()+"-----------------");
        return result;
    }
}
```

导出代理类：

```
public class CglibProxyTest {

    public static void main(String[] args){
        //生成代理类到本地
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "com/study/spring/proxy/cglib");
        UserServiceImpl service = new UserServiceImpl();
        CglibProxy cp = new CglibProxy();
        UserService proxy = (UserService) cp.getProxy(service.getClass());
        proxy.add();
    }
}
```

反编译代理类，由于代码有点长，因此删减了一部分：

```
public class UserServiceImpl$$EnhancerByCGLIB$$d166e24d extends UserServiceImpl implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$add$0$Method;
    private static final MethodProxy CGLIB$add$0$Proxy;

    final void CGLIB$add$0() {
        super.add();
    }

    public final void add() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if(this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if(var10000 != null) {
            var10000.intercept(this, CGLIB$add$0$Method, CGLIB$emptyArgs, CGLIB$add$0$Proxy);
        } else {
            super.add();
        }
    }
    public UserServiceImpl$$EnhancerByCGLIB$$d166e24d() {
        CGLIB$BIND_CALLBACKS(this);
    }

    public static void CGLIB$SET_THREAD_CALLBACKS(Callback[] var0) {
        CGLIB$THREAD_CALLBACKS.set(var0);
    }

    public static void CGLIB$SET_STATIC_CALLBACKS(Callback[] var0) {
        CGLIB$STATIC_CALLBACKS = var0;
    }

    private static final void CGLIB$BIND_CALLBACKS(Object var0) {
        UserServiceImpl$$EnhancerByCGLIB$$d166e24d var1 = (UserServiceImpl$$EnhancerByCGLIB$$d166e24d)var0;
        if(!var1.CGLIB$BOUND) {
            var1.CGLIB$BOUND = true;
            Object var10000 = CGLIB$THREAD_CALLBACKS.get();
            if(var10000 == null) {
                var10000 = CGLIB$STATIC_CALLBACKS;
                if(CGLIB$STATIC_CALLBACKS == null) {
                    return;
                }
            }

            var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
        }

    }

    public Object newInstance(Callback[] var1) {
        CGLIB$SET_THREAD_CALLBACKS(var1);
        UserServiceImpl$$EnhancerByCGLIB$$d166e24d var10000 = new UserServiceImpl$$EnhancerByCGLIB$$d166e24d();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Callback var1) {
        CGLIB$SET_THREAD_CALLBACKS(new Callback[]{var1});
        UserServiceImpl$$EnhancerByCGLIB$$d166e24d var10000 = new UserServiceImpl$$EnhancerByCGLIB$$d166e24d();
        CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
        return var10000;
    }

    public Object newInstance(Class[] var1, Object[] var2, Callback[] var3) {
        CGLIB$SET_THREAD_CALLBACKS(var3);
        UserServiceImpl$$EnhancerByCGLIB$$d166e24d var10000 = new UserServiceImpl$$EnhancerByCGLIB$$d166e24d;
        switch(var1.length) {
        case 0:
            var10000.<init>();
            CGLIB$SET_THREAD_CALLBACKS((Callback[])null);
            return var10000;
        default:
            throw new IllegalArgumentException("Constructor not found");
        }
    }

    public Callback getCallback(int var1) {
        CGLIB$BIND_CALLBACKS(this);
        MethodInterceptor var10000;
        switch(var1) {
        case 0:
            var10000 = this.CGLIB$CALLBACK_0;
            break;
        default:
            var10000 = null;
        }

        return var10000;
    }

    public void setCallback(int var1, Callback var2) {
        switch(var1) {
        case 0:
            this.CGLIB$CALLBACK_0 = (MethodInterceptor)var2;
        default:
        }
    }

    public Callback[] getCallbacks() {
        CGLIB$BIND_CALLBACKS(this);
        return new Callback[]{this.CGLIB$CALLBACK_0};
    }

    public void setCallbacks(Callback[] var1) {
        this.CGLIB$CALLBACK_0 = (MethodInterceptor)var1[0];
    }

    static {
        CGLIB$STATICHOOK1();
    }
}
```

通过代码我们知道，Cglib是通过直接继承被代理类

从代理类里面可知道对于原来的add函数，代理类里面对应了两个函数分别是add 和`CGLIB$add$0`
其中后者是在方法拦截器里面调用的，前者则是我们使用代理类时候调用的函数。当我们代码调用add时候，会具体调用到方法拦截器的intercept方法。

整体上JDK代理和CGLIB代理思想上都是差不多的，jdk基于接口，cglib基于继承（因此**方法不能被final修饰**）。

## 深入 AOP 源码实现

阅读源码很好用的一个方法就是跑代码来调试，因为自己一行一行地看的话，比较枯燥，而且难免会漏掉一些东西，为了更方便的调试代码，这里我们采用xml配置方式，简要的实现aop功能。

同样使用上面的 UserService 和UserServiceImpl类

创建一个Advice,这里实现 MethodBeforeAdvice接口：

```
public class BeforeAdvice implements MethodBeforeAdvice {

    public void before(Method method, Object[] objects, @Nullable Object o) throws Throwable{
        System.out.println(o.getClass() + ":"+method.getName()+" 方法准备执行");
    }
}
```

xml:

```
<bean id="userService" class="com.study.spring.aop.UserServiceImpl"/>
	<!--advice-->
	<bean id="beforeAdvice" class="com.study.spring.aop.BeforeAdvice" />
	<bean id="userServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean" >
        <!--代理接口类-->
		<property name="proxyInterfaces" value="com.study.spring.aop.UserService"/>
		<!--代理实现类-->
		<property name="target"  ref="userService"/>
		
		<!--走CGLIB代理-->
		<!--<property name="proxyTargetClass"  value="true"/>-->
		
		<!--配置拦截器,这里可以配置advice,advisor,interceptor-->
		<property name="interceptorNames">
			<!--引用我们beforeAdvice-->
			<list><value>beforeAdvice</value></list>
		</property>
</bean>
```

测试代码：

```
public void TestAOP(){

        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");

        ProxyFactoryBean abcProxy = (ProxyFactoryBean) context.getBean("&userServiceProxy");

        UserService userService= (UserService) abcProxy.getObject();

        userService.add();

        //实际获取的是被FactoryBean 包装后的bean
        userService = (UserService) context.getBean("userServiceProxy");

        userService.add();

}
```

运行结果这里就不展示了，在调用add 方法前，都会先执行我们的BeforeAdvice，AOP生效。

上面的代码中又涉及到了FactoryBean,这个我们在前面讲IOC的时候已经说过了，这里当然也就不在说了，不知道的朋友，可以自行再去看看。

### ProxyFactoryBean

代码很简单，配置也很简单，核心代码其实也就是`ProxyFactoryBean`,它是一个FactoryBean，那么在getBean的时候，会调用该FactoryBean的getObject 方法，因此 ProxyFactoryBean 的 getObject  方法就是我们的入口了。

#### getObject  

```
public Object getObject() throws BeansException {
	initializeAdvisorChain();
	if (isSingleton()) {
		return getSingletonInstance();
	}
	else {
		if (this.targetName == null) {
			logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
		}
		return newPrototypeInstance();
	}
}
```

该方法很重要，首先初始化Advice链，然后获取单例，这里返回的就是我们最初看到的代理bean。这里的初始化Advice链的重要作用就是将其连接起来，基本实现就是循环我们在配置文件中(interceptorNames)配置的拦截器，按照链表的方式连接起来。

#### initializeAdvisorChain

```
/**
* Create the advisor (interceptor) chain. Advisors that are sourced
* from a BeanFactory will be refreshed each time a new prototype instance
* is added. Interceptors added programmatically through the factory API
* are unaffected by such changes.
*/
private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
	if (this.advisorChainInitialized) {
		return;
	}
	if (!ObjectUtils.isEmpty(this.interceptorNames)) {
		if (this.beanFactory == null) {
			throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) " +
						"- cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
		}

		// Globals can't be last unless we specified a targetSource using the property...
	if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX) &&
			this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("Target required after globals");
		}
	for (String name : this.interceptorNames) {
		if (name.endsWith(GLOBAL_SUFFIX)) {
			if (!(this.beanFactory instanceof ListableBeanFactory)) {
				throw new AopConfigException(
								"Can only use global advisors or interceptors with a ListableBeanFactory");
			}
			addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
							name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
		}
		else {
			Object advice;
			if (this.singleton || this.beanFactory.isSingleton(name)) {
				//根据name 获取bean
				advice = this.beanFactory.getBean(name);
			}
				else {
					advice = new PrototypePlaceholderAdvisor(name);
				}
				addAdvisorOnChainCreation(advice, name);
			}
		}
	}

	this.advisorChainInitialized = true;
}
```

代码比较简单，核心思想就是初始化我们配置的拦截器，里面会递归调用getBean 方法。

我们着重关注下面的方法 getSingletonInstance()；

#### getSingletonInstance

```
private synchronized Object getSingletonInstance() {
	if (this.singletonInstance == null) {
		this.targetSource = freshTargetSource();
		if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
			// Rely on AOP infrastructure to tell us what interfaces to proxy.
			Class<?> targetClass = getTargetClass();
			if (targetClass == null) {
				throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
			}
			setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
		}
		// Initialize the shared singleton instance.
		super.setFrozen(this.freezeProxy);
		this.singletonInstance = getProxy(createAopProxy());
	}
	return this.singletonInstance;
}
```

该方法是同步方法，防止并发错误，因为有共享变量。首先返回一个包装过的目标对象，重要的一行是 `getProxy(createAopProxy())`，先创建AOP，再获取代理。我们先看 crateAopProxy。

```
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	return getAopProxyFactory().createAopProxy(this);
}
```

该方法返回一个AopProxy 类型的实例，我们看看该接口：

![20181031094421](http://img.blog.ztgreat.cn/document/spring/20181031094421.png)

这是该接口的继承图，分别是 JdkDynamicAopProxy 动态代理和 CglibAopProxy 代理。

我们继续看 createAopProxy 方法，该方法主要逻辑是创建一个AOP 工厂，默认工厂是 `DefaultAopProxyFactory`

#### DefaultAopProxyFactory

```
public ProxyCreatorSupport() {
   this.aopProxyFactory = new DefaultAopProxyFactory();
}
```



该类的 createAopProxy 方法则根据 ProxyFactoryBean 的一些属性来决定创建哪种代理：

```
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
		}
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			return new JdkDynamicAopProxy(config);
		}
		return new ObjenesisCglibAopProxy(config);
	}
	else {
		return new JdkDynamicAopProxy(config);
	}
}
```

到这里，我们知道 createAopProxy 方法有可能返回 JdkDynamicAopProxy 实例，也有可能返回 ObjenesisCglibAopProxy 实例，这里总结一下：

如果被代理的目标类实现了一个或多个自定义的接口，那么就会使用 JDK 动态代理，如果没有实现任何接口，会使用 CGLIB 实现代理，如果设置了 proxy-target-class="true"，那么通常都会使用 CGLIB。

**JDK 动态代理基于接口**，所以只有接口中的方法会被增强，而 **CGLIB 基于类继承，需要注意就是如果方法使用了 final 修饰，或者是 private 方法，是不能被增强的**。



我们分别来看下两个 AopProxy 实现类的 getProxy(classLoader) 实现。

### JdkDynamicAopProxy

##### getProxy

```
public Object getProxy(@Nullable ClassLoader classLoader) {
	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

java.lang.reflect.Proxy.newProxyInstance(…) 方法需要三个参数，第一个是 ClassLoader，第二个参数代表需要实现哪些接口，第三个参数最重要，是 InvocationHandler 实例，我们看到这里传了 this，因为 JdkDynamicAopProxy 本身实现了 InvocationHandler 接口。

```
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
	...
}
```



InvocationHandler 只有一个方法，当生成的代理类对外提供服务的时候，都会回到这个方法中：

##### invoke

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			// 代理equals 方法
			return equals(args[0]);
		}
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			// 代理hashCode 方法
			return hashCode();
		}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// There is only getDecoratedClass() declared -> dispatch to proxy config.
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;
        // 如果设置了 exposeProxy，那么将 proxy 放到 ThreadLocal 中
		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		// 创建一个 chain，包含所有要执行的 advice
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		
		if (chain.isEmpty()) {
			// chain 是空的，说明不需要被增强
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// We need to create a method invocation...
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			//执行增强，并调用方法
			retVal = invocation.proceed();
		}

		//返回值检查
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

核心逻辑不算复杂，重点分析创建advice chain,然后调用方法代码那里（44行~57行），可以进行断点调试，观察一下执行流程，这里就不继续追下去了，感兴趣的读者自己去深入探索下，不是很难。

说完了 JDK 动态代理 JdkDynamicAopProxy，我们再来看一下 CGLIB 的代理实现 ObjenesisCglibAopProxy。

ObjenesisCglibAopProxy 继承了 CglibAopProxy，而 CglibAopProxy 继承了 AopProxy。

###  CglibAopProxy

在前面我们知道得到AopProxy后，会调用getProxy 方法，因此这里我们直接就来看看该方法。

##### getProxy

```
public Object getProxy(@Nullable ClassLoader classLoader) {

	try {
		Class<?> rootClass = this.advised.getTargetClass();

		Class<?> proxySuperClass = rootClass;
		if (ClassUtils.isCglibProxyClass(rootClass)) {
			proxySuperClass = rootClass.getSuperclass();
			Class<?>[] additionalInterfaces = rootClass.getInterfaces();
			for (Class<?> additionalInterface : additionalInterfaces) {
				this.advised.addInterface(additionalInterface);
			}
		}

		// Validate the class, writing log messages as necessary.
		validateClassIfNecessary(proxySuperClass, classLoader);

		// Configure CGLIB Enhancer...
		Enhancer enhancer = createEnhancer();
		if (classLoader != null) {
			enhancer.setClassLoader(classLoader);
			if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
				enhancer.setUseCache(false);
			}
		}
		enhancer.setSuperclass(proxySuperClass);
		enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

		Callback[] callbacks = getCallbacks(rootClass);
		Class<?>[] types = new Class<?>[callbacks.length];
		for (int x = 0; x < types.length; x++) {
			types[x] = callbacks[x].getClass();
		}
		// fixedInterceptorMap only populated at this point, after getCallbacks call above
		enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
		enhancer.setCallbackTypes(types);

		// Generate the proxy class and create a proxy instance.
		return createProxyClassAndInstance(enhancer, callbacks);
	}
	catch (CodeGenerationException | IllegalArgumentException ex) {
		throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
					": Common causes of this problem include using a final class or a non-visible class",
					ex);
	}
	catch (Throwable ex) {
		// TargetSource.getTarget() failed
		throw new AopConfigException("Unexpected AOP exception", ex);
	}
}
```

从上面代码中，可以看到对CGLIB的使用，比如对Enhancer对象的配置，以及通过Enhancer 对象生成代理对象的过程。在这个生成代理对象的过程中，需要注意的是对Enhancer对象callback回调的设置，正是这些回调封装了Spring AOP的实现，就像前面介绍的JDK的Proxy对象的invoke回调方法一样。在Enhancer的callback回调设置中，实际上是通过设置DynamicAdvisedInterceptor拦截器来完成AOP功能的。

```
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
	// Parameters used for optimization choices...
	boolean exposeProxy = this.advised.isExposeProxy();
	boolean isFrozen = this.advised.isFrozen();
	boolean isStatic = this.advised.getTargetSource().isStatic();

	// Choose an "aop" interceptor (used for AOP calls).
	Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);
	...
}
```

DynamicAdvisedInterceptor 参数中的 this.advised 是一个`AdvisedSupport`实例，这个是在创建AOpProxy的时候传入的。

现在切入点在 DynamicAdvisedInterceptor  类了

##### DynamicAdvisedInterceptor

核心方法便是intercept 方法了，类似invoke 方法。

```
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;
	Object target = null;
	TargetSource targetSource = this.advised.getTargetSource();
	try {
		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}
		// Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);
		//回去拦截器链
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		Object retVal;
		// Check whether we only have one InvokerInterceptor: that is,
		// no real advice, but just reflective invocation of the target.
		if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
			//如果拦截器为空，则直接进行方法调用
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = methodProxy.invoke(target, argsToUse);
		}
		else {
			//拦截器调用，方法调用
			retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
		}
		retVal = processReturnType(proxy, target, method, retVal);
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

方法逻辑和JdkDynamicAopProxy 中的invoke 方法是差不多的，这里就不多说了。



## 总结

以ProxyFactoryBean为例，通过简单的配置，对Spring AOP的基本实现和工作流程进行了简单的梳理，通过ProxyFactoryBean 得到AopProxy对象。

通过使用AopProxy对象封装target目标对象之后，ProxyFactoryBean的getObject方法得到的对象就不是一个普通的Java对象了，而是一个AoppProxy代理对象。在ProxyFactoryBean中配置的target目标对象，这时已经不会让应用直接调用其方法实现，而是作为AOP实现的一部分。对target目标对象的方法调用会首先被AopProxy代理对象拦截，对于不同的AopProxy代理对象生成方法，会使用不同的拦截回调入口。例如，对于JDK的AopProxy代理对象，使用是InvocationHandler的invoke回调入口；而对于CGLIB的AopProxy代理对象，使用的是设置好的callback回调，在这些callback 回调中，对于AOP实现，是通过DynamicAdvisedInterceptor 来完成的，而DynamicAdvisedInterceptor的回调入口是intercept方法。

## 参考

Spring 技术内幕

[JDK动态代理代理与Cglib代理原理探究](http://ifeve.com/jdk%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E4%BB%A3%E7%90%86%E4%B8%8Ecglib%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/)