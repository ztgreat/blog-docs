# dubbo 启动概述(一)－服务加载



## 前言

在项目启动过程中，`dubbo服务`如何随项目的启动而发布？dubbo如何随着`spring容器`的初始化而启动?

我们使用的dubbo 注解`@Service` 是如何工作的(或者xml 配置)，下面我们将来分析dubbo 服务端是如何随着Spring 启动而启动的，本文主要分析基于注解方式。



本文需要**Spring 启动流程加载**的一些知识，这里不会细讲,具体可以参考我以前写的[Spring IOC](http://blog.ztgreat.cn/article/57)相关的知识



## dubbo 标签解析

`Dubbo`基于`Spring`提供的`NamespaceHandler`和`BeanDefinitionParser`来扩展了自己`XML Schemas`，一切的功能也都是基于这一点来进行的。



![20191126_214710.png](http://img.blog.ztgreat.cn/document/dubbo/20191126_214710.png)



可以看到　图中，`dubbo-config` 模块，resources 下面有一系列的文件，Dubbo 中的自定义 XML 标签实际上是依赖于 Spring 解析自定义标签的功能实现的，这里我们仅介绍下实现相关功能需要的文件，给大家一个直观的印象，不去深入的对 Spring 自定义标签实现作详细分析。

1. **定义 xsd 文件**
    XSD(XML Schemas Definition) 即 XML 结构定义。我们通过 XSD 文件不仅可以定义新的元素和属性，同时也使用它对我们的 XML 文件规范进行约束。 在 Dubbo 项目中可以找类似实现：**dubbo.xsd**
2. **spring.schemas**
    该配置文件约定了自定义命名空间和 xsd 文件之间的映射关系，用于 spring 容器感知我们自定义的 xsd 文件位置。

```
http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
```

1. **spring.handlers**
    该配置文件约定了自定义命名空间和 NamespaceHandler 类之间的映射关系。 NamespaceHandler 类用于注册自定义标签解析器。

```
http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
```

1. **命名空间处理器**
    命名空间处理器主要用来注册 BeanDefinitionParser 解析器。对应上面 spring.handlers 文件中的 DubboNamespaceHandler

```
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        // 省略...
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```

1. **BeanDefinitionParser 解析器**
    实现 BeanDefinitionParser 接口中的 parse 方法，用于自定义标签的解析。Dubbo 中对应 DubboBeanDefinitionParser 类。





上述　是xml 的配置的解析，如果我们使用注解　这种怎么处理呢，下面我们看看**基于注解**的相关流程。



## 注册@Service bean处理器



### Bean 注册后置处理器



首先　我们　需要先来了解一下　`BeanDefinitionRegistryPostProcessor`，这个接口扩展自BeanFactoryPostProcessor，专门用于动态注册Bean，dubbo 就是　利用　这个　扩展　来注册自己相关的一些服务。



![20191126_215239.png](http://img.blog.ztgreat.cn/document/dubbo/20191126_215239.png)



### ServiceBean 后置处理器

接下来　我们需要知道　`ServiceAnnotationBeanPostProcessor`　这个　处理器，我们先看一下继承关系



![20191126_224332.png](http://img.blog.ztgreat.cn/document/dubbo/20191126_224332.png)



这里可以看到ServiceAnnotationBeanPostProcessor实现了BeanDefinitionRegistryPostProcessor接口的方法，postProcessBeanDefinitionRegistry方法传入BeanDefinitionRegistry实例

该方法是将ServiceBean的BeanDefinition注册到BeanDefinitiond到BeanDefinitionRegistry实例中.



> ServiceBean 是指 含有 dubbo  @Service 注解的bean



```
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {

    private final Set<String> packagesToScan;

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            // 注册 含有 dubbo  @Service 注解的bean
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
 
    //... 省略其它代码
 }
```



这里打断一下，这个　ServiceAnnotationBeanPostProcessor 又是怎么被加载呢？？

ServiceAnnotationBeanPostProcessor 是被 DubboComponentScanRegistrar 加载 进来的



```
/**
 * Dubbo {@link DubboComponentScan} Bean Registrar
 *
 * @see Service
 * @see DubboComponentScan
 * @see ImportBeanDefinitionRegistrar
 * @see ServiceAnnotationBeanPostProcessor
 * @see ReferenceAnnotationBeanPostProcessor
 * @since 2.5.7
 */
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);

        registerReferenceAnnotationBeanPostProcessor(registry);

    }

    /**
     * Registers {@link ServiceAnnotationBeanPostProcessor}
     *
     * @param packagesToScan packages to scan without resolving placeholders
     * @param registry       {@link BeanDefinitionRegistry}
     * @since 2.5.8
     */
    private void registerServiceAnnotationBeanPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationBeanPostProcessor.class);
        builder.addConstructorArgValue(packagesToScan);
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);

    }

    /**
     * Registers {@link ReferenceAnnotationBeanPostProcessor} into {@link BeanFactory}
     *
     * @param registry {@link BeanDefinitionRegistry}
     */
    private void registerReferenceAnnotationBeanPostProcessor(BeanDefinitionRegistry registry) {

        // Register @Reference Annotation Bean Processor
        BeanRegistrar.registerInfrastructureBean(registry,
                ReferenceAnnotationBeanPostProcessor.BEAN_NAME, ReferenceAnnotationBeanPostProcessor.class);

    }

}
```



同样的　那么　DubboComponentScanRegistrar 又是怎么样被加载进来的呢？？

这里就要看几个注解了：`*DubboComponentScan*`*，*`EnableDubbo`，这里就提一下，作为一个简单的介绍。



![20191126_223238.png](http://img.blog.ztgreat.cn/document/dubbo/20191126_223238.png)



![20191126_223359.png](http://img.blog.ztgreat.cn/document/dubbo/20191126_223359.png)

ok,回来　我们　继续看我们的ServiceBean相关的问题

## ServiceBean

### ServiceBean是什么

服务暴露是由com.alibaba.dubbo.config.spring.ServiceBean这个类来实现的，这个类是spring通过解析dubbo:service节点创建的单例Bean，每一个dubbo:service都会创建一个ServiceBean,每个　使用了dubbo 的`＠``Service`注解的 bean 也是一个ServiceBean,ServiceBean 是 Dubbo 与 Spring 框架进行整合的关键，可以看做是两个框架之间的桥梁。具有同样作用的类还有 ReferenceBean。



![20191127_205309.png](http://img.blog.ztgreat.cn/document/dubbo/20191127_205309.png)





```
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,
        ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,
        ApplicationEventPublisherAware {


    private static final long serialVersionUID = 213195494150089726L;

    private final transient Service service;

    private transient ApplicationContext applicationContext;

    private transient String beanName;

    private transient boolean supportedApplicationListener;

    private ApplicationEventPublisher applicationEventPublisher;

    public ServiceBean() {
        super();
        this.service = null;
    }
    // ... 省略其它代码
}
```



从上面类图可以看到实现了InitializingBean和ApplicationListener

所以afterPropertiesSet和onApplicationEvent是ServiceBean的入口,这里暂时不会展开讨论，服务端有ServiceBean，消费端来说　当然有ReferenceBean，下面有简单介绍ReferenceBean，可以提前跳下去看看。不过。



### 导出 ServiceBean

当我们　实例化了一个被dubbo `@Service`　标注的　bean 后，就要考虑消费端如何访问了，因此这个时候就需要把我们的ServiceBean　进行暴露。

> Dubbo 服务导出过程始于 Spring 容器发布刷新事件，Dubbo 在接收到事件后，会立即执行服务导出逻辑。整个逻辑大致可分为三个部分：
>
> 第一部分是前置工作，主要用于检查参数，组装 URL。
>
> 第二部分是导出服务，包含导出服务到本地 (JVM)，和导出服务到远程两个过程。
>
> 第三部分是向注册中心注册服务，用于服务发现



从Dubbo官网上找到一张服务暴露的时序图，可以先了解一下，有个印象。

![1663e55f218fe08b.png](http://img.blog.ztgreat.cn/document/dubbo/1663e55f218fe08b.png)



服务导出的入口方法是 ServiceBean 的 onApplicationEvent。onApplicationEvent 是一个事件响应方法，该方法会在收到 Spring 上下文刷新事件后执行服务导出操作。方法代码如下：

```
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 是否有延迟导出 && 是否已导出 && 是不是已被取消导出
    if (isDelay() && !isExported() && !isUnexported()) {
        // 导出服务
        export();
    }
}
```

这个方法首先会根据条件决定是否导出服务，比如有些服务设置了延时导出，那么此时就不应该在此处导出。还有一些服务已经被导出了，或者当前服务被取消导出了，此时也不能再次导出相关服务。注意这里的 isDelay 方法，这个方法字面意思是“是否延迟导出服务”，返回 true 表示延迟导出，false 表示不延迟导出。但是该方法真实意思却并非如此，当方法返回 true 时，表示无需延迟导出。返回 false 时，表示需要延迟导出。与字面意思恰恰相反，这个需要大家注意一下。

好，具体的导出服务这里暂时不讲，先了解一个大概，后面再来看看　具体的导出服务。



## ReferenceBean

同样的　跟服务引用一样，Dubbo的reference配置会被转成ReferenceBean类.



![20191127_222730.png](http://img.blog.ztgreat.cn/document/dubbo/20191127_222730.png)



这里　我们　需要注意的是ReferenceBean　其实是一个FactoryBean。

> FactoryBean：是一个Java Bean，但是它是一个能生产对象的工厂Bean，它的实现和工厂模式及修饰器模式很像



这里不多讲，感兴趣的朋友自己去了解，或者看我以前写的Spring IOC相关的解析，这次我们的主题主要还是围绕ServiceBean来展开。



## 总结

1. dubbo 的标签是怎么让spring 解析的
2. 基于注册的dubbo 相关的类　是怎么被spring加载进来的
3. 被dubbo ＠Service 注解的类到底是怎么样的一个bean
4. ServiceBean 是怎么加载的