## 前言

前面分析了Spring BeanFactory,接着分析了Spring IOC的初始化过程，对整个流程有了一定的认识，当然没有面面俱到，当然也不可能，我自己本身定位就是把握主要脉络，前面遗留了一个问题，就是在Spring IOC最后初始化单例bean的时候，针对循环依赖的处理问题，学习一下思想，处理方式，这是这篇文章的主要目的。

## 循环依赖？

循环依赖其实就是循环引用，也就是两个或则两个以上的bean互相持有对方，最终形成闭环。比如A依赖于B，B依赖于C，C又依赖于A。

Spring中循环依赖场景有： 

（1）构造器的循环依赖 （没法解决）

（2）field属性的循环依赖。

## 是否存在循环依赖

我们知道Spring 在创建Bean的时候如果需要创建依赖Bean，则会进行递归调用，那么这样检测循环依赖相对比较容易了，Bean在创建的时候可以给该Bean打个标记，如果递归调用回来发现正在创建中的话，即说明了循环依赖了（是不是有种感觉树的递归遍历）。

## Spring怎么解决循环依赖

Spring 在处理属性循环依赖的情况时主要是通过延迟设置来解决的，当bean被实例化后，此时还没有进行依赖注入，当进行依赖注入的时候，发现依赖的bean已经处于创建中了，那么通过可以设置依赖，虽然依赖的bean没有初始化完成，但是这个可以后面再进行设置，至少得先建立bean之间的引用。

Spring的单例对象的初始化主要分为三步： 

（1）createBeanInstance：实例化

（2）populateBean：填充属性

（3）initializeBean：调用spring xml中的init 方法。

第一步存在构造器循环依赖

第二步存在属性循环依赖

那么我们要解决循环引用也应该从初始化过程着手，对于单例来说，在Spring容器整个生命周期内，有且只有一个对象，所以很容易想到这个对象应该存在Cache中

Spring为了解决单例的循环依赖问题，使用了三级缓存。

首先我们看源码，三级缓存主要指：

`DefaultSingletonBeanRegistry:`

```
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

```

这三级缓存分别指： 
singletonFactories ： 单例对象工厂的cache 
earlySingletonObjects ：提前暴光的单例对象的Cache 
singletonObjects：单例对象的cache

我们在创建bean的时候，首先是从cache中获取这个单例的bean，这个缓存就是singletonObjects。主要调用方法就就是：

```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //获取bean
    Object singletonObject = this.singletonObjects.get(beanName);
    
    //如果该bean在创建中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            //从提前曝光的集合中获取bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            
            //没找到，允许提前引用
            if (singletonObject == null && allowEarlyReference) {
                获取factory
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    //通过factory 获取bean
                    singletonObject = singletonFactory.getObject();
                    
                    //提前曝光
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}

```


上面的代码需要解释两个参数：

isSingletonCurrentlyInCreation()判断当前单例bean是否正在创建中，也就是没有初始化完成(比如在A的populateBean过程中依赖了B对象，得先去创建B对象，这时的A就是处于创建中的状态。

allowEarlyReference 是否允许从singletonFactories中通过getObject拿到对象

getSingleton()的整个过程，Spring首先从一级缓存singletonObjects中获取。如果获取不到，并且对象正在创建中，就再从二级缓存earlySingletonObjects中获取。如果还是获取不到且允许singletonFactories通过getObject()获取，就从三级缓存singletonFactory.getObject()(三级缓存)获取

如果获取到了则：

```
this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
```

从singletonFactories中移除，并放入earlySingletonObjects中。其实也就是从三级缓存移动到了二级缓存。

从上面三级缓存的分析，我们可以知道，Spring解决循环依赖的诀窍就在于singletonFactories这个三级cache。

在上篇文章中，我们分析了Spring IOC的初始化过程，我们再来回顾一下 `AbstractAutowireCapableBeanFactory`中的`doCreateBean`方法：

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
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		//提前曝光bean		
		if (earlySingletonExposure) {
		    //这里是重点
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

核心代码是`addSingletonFactory`方法，该方法介于bean实例化和依赖注入（populateBean）中间，这个对象已经被生产出来了，虽然还没有真正初始化，但是可以根据对象引用能定位到堆中的对象，所以Spring此时将这个对象提前曝光出来让大家认识，让大家使用。

```
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
}
```

如果容器允许提前曝光和引用，那么在singletonFactories 中放入该bean的singletonFactory，这样在依赖注入的时候，就能在第三级缓存中找到singletonFactory 了。

经过这样的操作后，面对 "A 依赖了B，同时B依赖了A"这种循环依赖的情况。A首先完成了初始化的第一步，并且将自己提前曝光到singletonFactories中，此时进行初始化的第二步（依赖注入），发现自己依赖对象B，此时就尝试去get(B)，发现B还没有被create，所以走create流程，B在初始化第一步的时候发现自己依赖了对象A，于是尝试get(A)，尝试一级缓存singletonObjects(肯定没有，因为A还没初始化完全)，尝试二级缓存earlySingletonObjects（也没有），尝试三级缓存singletonFactories，由于A通过ObjectFactory将自己提前曝光了，因此B能够通过ObjectFactory.getObject拿到A对象(虽然A还没有初始化完全，但是总比没有好呀)，B拿到A对象后顺利完成了初始化阶段1、2、3，完全初始化之后将自己放入到一级缓存singletonObjects中。此时返回A中，A此时能拿到B的对象顺利完成自己的初始化阶段2、3，最终A也完成了初始化，进去了一级缓存singletonObjects中，而且更加幸运的是，由于B拿到了A的对象引用，所以B现在hold住的A对象完成了初始化，最后皆大欢喜，A,B都成功的初始化了。

现在 知道为什么 Spring不能解决"A的构造方法中依赖了B的实例对象，同时B的构造方法中依赖了A的实例对象"这类问题了,因为加入singletonFactories三级缓存的**前提是执行了构造器，所以构造器的循环依赖没法解决**。



## 特殊情况说明

前面说到Spring 利用三级缓存解决了Bean属性循环依赖问题，这里面有个细节问题，接着上面的例子来讲：如果在创建B时，如果调用了A的一些方法，那么就存在危险，因为此时A的属性还未完成初始化，这种情况在spring 配置中指定B的init 方法，这样bean属性注入后会调用init方法，下面简单演示一下这种情况，当然很少会这样做，这里只是钻空子罢了。

A对象：

```
public class CycleA {

    //依赖B
    private  CycleB cycleB;
    //依赖bean
    private MyBean bean;

    public CycleB getCycleB() {
        return cycleB;
    }

    public void setCycleB(CycleB cycleB) {
        this.cycleB = cycleB;
    }

    public MyBean getBean() {
        return bean;
    }

    public void setBean(MyBean bean) {
        this.bean = bean;
    }

    @Override
    public String toString() {
        return "this is CycleA";
    }
}
```

B对象

```
public class CycleB {

    //依赖A
    private  CycleA cycleA;

    public CycleA getCycleA() {
        return cycleA;
    }

    public void setCycleA(CycleA cycleA) {
        this.cycleA = cycleA;
    }
    //init方法中，调用A获取bean属性
    public  void init(){
        System.out.println("cycleA:"+cycleA.toString());
        //cycleA.getBean() 为 null 
        System.out.println("cycleA->bean:"+cycleA.getBean());
    }

    @Override
    public String toString() {
        return "this is CycleB";
    }
}
```

MyBean对象：

```
public class MyBean {

    public void init(){
        System.out.println("MyBean:this is init method");
    }
    @Override
    public String toString() {
        return "this is My Bean";
    }
}
```

Spring xml配置

```
	<bean id="myBean" class="com.study.spring.bean.MyBean" />
    <bean id="cycleA" class="com.study.spring.bean.CycleA" scope="singleton">
        <property  name="cycleB" ref="cycleB"></property>
        <property  name="bean" ref="myBean"></property>

    </bean>
    <bean id="cycleB" class="com.study.spring.bean.CycleB" scope="singleton" init-method="init">
        <property  name="cycleA" ref="cycleA"></property>
    </bean>
```

结果：

![20181023155551](http://img.blog.ztgreat.cn/document/spring/20181023155551.png)

在Bean对象中虽然可以获取A对象，但是A对象的属性还为注入完毕，没有进行一系列初始化，所以是不安全的。

最后Spring 这种处理循环依赖的这种思想值得我们学习借鉴。

## 参考

[Spring-bean的循环依赖以及解决方式](https://blog.csdn.net/u010853261/article/details/77940767)	