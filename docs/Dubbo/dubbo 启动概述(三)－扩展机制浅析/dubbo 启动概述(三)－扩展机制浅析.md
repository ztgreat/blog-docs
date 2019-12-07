## 前言

在上篇文章，我们了解了Dubbo 服务端的暴露流程，在最后，提出关于一行代码的疑问



```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

不管是暴露到本地，还是暴露到远程，最后都是调用的**protocol.export，**但是发现不同时候具体的protocol 是不一样的，今天我们就来了解背后的逻辑，这里牵涉出了dubbo 的扩展机制。

可以参考官网的文章：[Dubbo可扩展机制实战](http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html)，本文也大量参考官网的文档，然后结合了自己的一些理解。



## 1.Dubbo的扩展机制

在谈到软件设计时，可扩展性一直被谈起，那到底什么才是可扩展性，什么样的框架才算有良好的可扩展性呢？它必须要做到以下两点:

1. 作为框架的维护者，在添加一个新功能时，只需要添加一些新代码，而不用大量的修改现有的代码，即符合开闭原则。
2. 作为框架的使用者，在添加一个新功能时，不需要去修改框架的源码，在自己的工程中添加代码即可。

Dubbo很好的做到了上面两点。这要得益于Dubbo的微内核+插件的机制。接下来的章节中我们会慢慢揭开Dubbo扩展机制的神秘面纱。

## 2. 可扩展的几种解决方案

通常可扩展的实现有下面几种:

- Factory模式
- IoC容器
- OSGI容器

Dubbo作为一个框架，不希望强依赖其他的IoC容器，比如Spring，Guice。OSGI也是一个很重的实现，不适合Dubbo。最终Dubbo的实现参考了Java原生的SPI机制，但对其进行了一些扩展，以满足Dubbo的需求。

## 3. Java SPI机制

既然Dubbo的扩展机制是基于Java原生的SPI机制，那么我们就先来了解下Java SPI吧。了解了Java的SPI，也就是对Dubbo的扩展机制有一个基本的了解。如果对Java SPI比较了解的同学，可以跳过。

Java SPI(Service Provider Interface)是JDK内置的一种动态加载扩展点的实现。在ClassPath的`META-INF/services`目录下放置一个与接口同名的文本文件，文件的内容为接口的实现类，多个实现类用换行符分隔。JDK中使用`java.util.ServiceLoader`来加载具体的实现。 让我们通过一个简单的例子，来看看Java SPI是如何工作的。

1. 定义一个接口IRepository用于实现数据储存

```
public interface IRepository {
    void save(String data);
}
```

1. 提供IRepository的实现 IRepository有两个实现。MysqlRepository和MongoRepository。

```
public class MysqlRepository implements IRepository {
    public void save(String data) {
        System.out.println("Save " + data + " to Mysql");
    }
}
public class MongoRepository implements IRepository {
    public void save(String data) {
        System.out.println("Save " + data + " to Mongo");
    }
}
```

1. 添加配置文件 在`META-INF/services`目录添加一个文件，文件名和接口全名称相同，所以文件是`META-INF/services/com.demo.IRepository`。文件内容为:

```
com.demo.MongoRepository
com.demo.MysqlRepository
```

1. 通过ServiceLoader加载IRepository实现

```
ServiceLoader<IRepository> serviceLoader = ServiceLoader.load(IRepository.class);
Iterator<IRepository> it = serviceLoader.iterator();
while (it != null && it.hasNext()){
    IRepository demoService = it.next();
    System.out.println("class:" + demoService.getClass().getName());
    demoService.save("tom");
}
```

在上面的例子中，我们定义了一个扩展点和它的两个实现。在ClassPath中添加了扩展的配置文件，最后使用ServiceLoader来加载所有的扩展点。 最终的输出结果为： class:testDubbo.MongoRepository Save tom to Mongo class:testDubbo.MysqlRepository Save tom to Mysql

## 4. Dubbo的SPI机制

Java SPI的使用很简单。也做到了基本的加载扩展点的功能。但Java SPI有以下的不足:

- 需要遍历所有的实现，并实例化，然后我们在循环中才能找到我们需要的实现。
- 配置文件中只是简单的列出了所有的扩展实现，而没有给他们命名。导致在程序中很难去准确的引用它们。
- 扩展如果依赖其他的扩展，做不到自动注入和装配
- 不提供类似于Spring的IOC和AOP功能
- 扩展很难和其他的框架集成，比如扩展里面依赖了一个Spring bean，原生的Java SPI不支持

所以Java SPI应付一些简单的场景是可以的，但对于Dubbo，它的功能还是比较弱的。Dubbo对原生SPI机制进行了一些扩展。接下来，我们就更深入地了解下Dubbo的SPI机制。

## 5. Dubbo扩展点机制基本概念

在深入学习Dubbo的扩展机制之前，我们先明确Dubbo SPI中的一些基本概念。在接下来的内容中，我们会多次用到这些术语。

#### 5.1 扩展点(Extension Point)

是一个Java的接口。

#### 5.2 扩展(Extension)

扩展点的实现类。

#### 5.3 扩展实例(Extension Instance)

扩展点实现类的实例。

#### 5.4 扩展自适应实例(Extension Adaptive Instance)

第一次接触这个概念时，可能不太好理解(我第一次也是这样的...)。如果称它为扩展代理类，可能更好理解些。扩展的自适应实例其实就是一个Extension的代理，它实现了扩展点接口。在调用扩展点的接口方法时，会根据实际的参数来决定要使用哪个扩展。比如一个IRepository的扩展点，有一个save方法。有两个实现MysqlRepository和MongoRepository。IRepository的自适应实例在调用接口方法的时候，会根据save方法中的参数，来决定要调用哪个IRepository的实现。如果方法参数中有repository=mysql，那么就调用MysqlRepository的save方法。如果repository=mongo，就调用MongoRepository的save方法。和面向对象的延迟绑定很类似。为什么Dubbo会引入扩展自适应实例的概念呢？

- Dubbo中的配置有两种，一种是固定的系统级别的配置，在Dubbo启动之后就不会再改了。还有一种是运行时的配置，可能对于每一次的RPC，这些配置都不同。比如在xml文件中配置了超时时间是10秒钟，这个配置在Dubbo启动之后，就不会改变了。但针对某一次的RPC调用，可以设置它的超时时间是30秒钟，以覆盖系统级别的配置。对于Dubbo而言，每一次的RPC调用的参数都是未知的。只有在运行时，根据这些参数才能做出正确的决定。
- 很多时候，我们的类都是一个单例的，比如Spring的bean，在Spring bean都实例化时，如果它依赖某个扩展点，但是在bean实例化时，是不知道究竟该使用哪个具体的扩展实现的。这时候就需要一个代理模式了，它实现了扩展点接口，方法内部可以根据运行时参数，动态的选择合适的扩展实现。而这个代理就是自适应实例。 自适应扩展实例在Dubbo中的使用非常广泛，Dubbo中，每一个扩展都会有一个自适应类，如果我们没有提供，Dubbo会使用字节码工具为我们自动生成一个。所以我们基本感觉不到自适应类的存在。后面会有例子说明自适应类是怎么工作的。

#### 5.5 @SPI

@SPI注解作用于扩展点的接口上，表明该接口是一个扩展点。可以被Dubbo的ExtentionLoader加载。如果没有此ExtensionLoader调用会异常。

#### 5.6 @Adaptive

@Adaptive注解用在扩展接口的方法上。表示该方法是一个自适应方法。Dubbo在为扩展点生成自适应实例时，如果方法有@Adaptive注解，会为该方法生成对应的代码。方法内部会根据方法的参数，来决定使用哪个扩展。 @Adaptive注解用在类上代表实现一个装饰类，类似于设计模式中的装饰模式，它主要作用是返回指定类，目前在整个系统中AdaptiveCompiler、AdaptiveExtensionFactory这两个类拥有该注解。

#### 5.7 ExtentionLoader

类似于Java SPI的ServiceLoader，负责扩展的加载和生命周期维护。

#### 5.8 扩展别名

和Java SPI不同，Dubbo中的扩展都有一个别名，用于在应用中引用它们。比如

```
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
registry=org.apache.dubbo.registry.integration.RegistryProtocol
injvm=org.apache.dubbo.rpc.protocol.injvm.InjvmProtocol
```

其中的dubbo，registry,injvm就是对应扩展的别名。这样我们在配置文件中使用dubbo或registry就可以了。

#### 5.9 一些路径

和Java SPI从`/META-INF/services`目录加载扩展配置类似，Dubbo也会从以下路径去加载扩展配置文件:

- `META-INF/dubbo/internal`
- `META-INF/dubbo`
- `META-INF/services`

## 6. LoadBalance扩展点解读

在了解了Dubbo的一些基本概念后，让我们一起来看一个Dubbo中实际的扩展点，对这些概念有一个更直观的认识。

我们选择的是Dubbo中的LoadBalance扩展点。Dubbo中的一个服务，通常有多个Provider，consumer调用服务时，需要在多个Provider中选择一个。这就是一个LoadBalance。我们一起来看看在Dubbo中，LoadBalance是如何成为一个扩展点的。

### 6.1 LoadBalance接口

```
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

LoadBalance接口只有一个select方法。select方法从多个invoker中选择其中一个。上面代码中和Dubbo SPI相关的元素有:

- @SPI([RandomLoadBalance.NAME](http://randomloadbalance.name/)) @SPI作用于LoadBalance接口，表示接口LoadBalance是一个扩展点。如果没有@SPI注解，试图去加载扩展时，会抛出异常。@SPI注解有一个参数，该参数表示该扩展点的默认实现的别名。如果没有显示的指定扩展，就使用默认实现。`RandomLoadBalance.NAME`是一个常量，值是"random"，是一个随机负载均衡的实现。 random的定义在配置文件`META-INF/dubbo/internal/com.alibaba.dubbo.rpc.cluster.LoadBalance`中:

```
random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
leastactive=com.alibaba.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
consistenthash=com.alibaba.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
```

可以看到文件中定义了4个LoadBalance的扩展实现。由于负载均衡的实现不是本次的内容，这里就不过多说明。只用知道Dubbo提供了4种负载均衡的实现，我们可以通过xml文件，properties文件，JVM参数显式的指定一个实现。如果没有，默认使用随机。



- @Adaptive("loadbalance") @Adaptive注解修饰select方法，表明方法select方法是一个可自适应的方法。Dubbo会自动生成该方法对应的代码。当调用select方法时，会根据具体的方法参数来决定调用哪个扩展实现的select方法。@Adaptive注解的参数`loadbalance`表示方法参数中的loadbalance的值作为实际要调用的扩展实例。 但奇怪的是，我们发现select的方法中并没有loadbalance参数，那怎么获取loadbalance的值呢？select方法中还有一个URL类型的参数，Dubbo就是从URL中获取loadbalance的值的。这里涉及到Dubbo的URL总线模式，简单说，URL中包含了RPC调用中的所有参数。URL类中有一个`Map parameters`字段，parameters中就包含了loadbalance。

### 6.2 获取LoadBalance扩展

Dubbo中获取LoadBalance的代码如下:

```
LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName);
```

使用ExtensionLoader.getExtensionLoader(LoadBalance.class)方法获取一个ExtensionLoader的实例，然后调用getExtension，传入一个扩展的别名来获取对应的扩展实例。



## 7.服务暴露扩展点解读

前面的内容基本上都是官网的内容，我觉得基本上还是说得很清楚，下面我们在上一篇文章(服务暴露)的基础上扩展和深入，加深理解。

在前面服务暴露过程中有这么一段代码：



```
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    
    // 省略无关代码
    
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        // 加载 ConfiguratorFactory，并生成 Configurator 实例，然后通过实例配置 url
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    // 如果 scope = none，则什么都不做
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
        // scope != remote，导出到本地
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }

        // scope != local，导出到远程
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            //暂时先省略
        }
    }
    this.urls.add(url);
}
```



### 7.1 本地导出

我们还是来看如何把服务导出到本地的：**exportLocal方法。**



```
private void exportLocal(URL url) {
        URL local = URLBuilder.from(url)
                .setProtocol(LOCAL_PROTOCOL)
                .setHost(LOCALHOST_VALUE)
                .setPort(0)
                .build();
        Invoker invoker=PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local);
        Exporter<?> exporter = protocol.export(invoker);
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
}
```



getInvoker方法我们暂不关注。



```
@SPI("dubbo")
public interface Protocol {

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    
    // ...
}
```



我们看到export　方法被`@Adaptive`标注，那么这个会从invoker中获取URL,然后在从URL 中获取protocol 类型，取参数这个过程是被dubbo 代理了，这里的protocol是`P``rotocol$Adaptive`,这个也是自动生成的，这里的话，贴一下生成的代码(去掉了一些异常判断)，具体的不多讲，更多的可以看一下`ExtensionLoader`相关的方法。



```
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
      
        //从　Invoker 中获取URL
        org.apache.dubbo.common.URL url = arg0.getUrl();
        //获取扩展名
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        //加载扩展类
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        //　export 委托给具体的扩展类
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public void destroy() {
        throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }
}
```



相关的注释写在代码里面了，很简单，下面我们关注**protocol.export**，我们看一下**exportLocal**中invoker的URL信息。

下面是我本机测试时候的URL:



```
injvm://127.0.0.1/org.apache.dubbo.demo.DemoService?
anyhost=true&application=dubbo-demo-annotation-provider&
bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&
bind.ip=192.168.0.58&bind.port=20880&deprecated=false&dubbo=2.0.2&
dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&
methods=sayHello,sayOK&pid=14948&release=&
side=provider&timestamp=1575448906108
```



这里的protocol 就是injvm，暴露到本地，也就是jvm 中，既然如此，那么我们就到protocol 的子类`InjvmProtocol`看看：



```
public class InjvmProtocol extends AbstractProtocol implements Protocol {

    public static final String NAME = LOCAL_PROTOCOL;

    public static final int DEFAULT_PORT = 0;
    private static InjvmProtocol INSTANCE;

    public InjvmProtocol() {
        INSTANCE = this;
    }
    public static InjvmProtocol getInjvmProtocol() {
        if (INSTANCE == null) {
            ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(InjvmProtocol.NAME); // load
        }
        return INSTANCE;
    }

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        //　返回了　InjvmExporter
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }
    // 省略其它方法
}
```



看到这里应该就很明白了，通过我们的URL 获取到具体的Protocol，然后再委托给具体的Protocol 实现类来做导出任务。

下面我们做个简单的小调整，来扩展一下。



```
private void exportLocal(URL url) {
        URL local = URLBuilder.from(url)
                        // 我们把protocol 改成hello
                .setProtocol("hello")
                .setHost(LOCALHOST_VALUE)
                .setPort(0)
                .build();
        Invoker invoker=PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local);
        Exporter<?> exporter = protocol.export(invoker);
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
 }
```



接下来，我们定一个HelloProtocol:



```
public class HelloProtocol extends AbstractProtocol implements Protocol {

    public static final String NAME = "hello";

    public static final int DEFAULT_PORT = 0;
    private static HelloProtocol INSTANCE;

    public HelloProtocol() {
        INSTANCE = this;
    }

    public static HelloProtocol getInjvmProtocol() {
        if (INSTANCE == null) {
            ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(HelloProtocol.NAME); // load
        }
        return INSTANCE;
    }

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        //这里我们可以自定义
        System.out.println("this is hello protol,哈哈哈哈哈哈哈哈");
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }
    //其它省略
}
```



![20191204_172010.png](http://img.blog.ztgreat.cn/document/dubbo/20191204_172010.png)

这样在导出服务到本地的时候，就会加载　`HelloProtocol`，可以自己多多尝试，加深理解。

### 7.2 导出到远程

同样的，在导出到远程的时候，我们查看一下URL,协议头是registry.



```
registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?
application=dubbo-demo-annotation-provider&dubbo=2.0.2&
export=dubbo%3A%2F%2F192.168.0.57%3A20880%2Forg.apache.dubbo.demo.DemoService%3F
anyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26
bean.name%3DServiceBean%3Aorg.apache.dubbo.demo.DemoService%26
bind.ip%3D192.168.0.57%26bind.port%3D20880%26deprecated%3Dfalse%26
dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26
interface%3Dorg.apache.dubbo.demo.DemoService%26
methods%3DsayHello%2CsayOK%26pid%3D18661%26release%3D%26
side%3Dprovider%26timestamp%3D1575118185436&pid=18661&
registry=zookeeper&timestamp=1575118181384
```



那么我们就可以去看 dubbo 中关于registry的配置，这样我们就可以方便的调试了，同时在上篇文章中，我们是跳过了这部分的介绍，直接定位到`RegistryProtocol`中的代码。这里我们就和前面衔接起来了。

```
registry=org.apache.dubbo.registry.integration.RegistryProtocol
```



这样通过dubbo 的SPI 机制可以很方便的进行扩展，同时对我们自身来说，也是一个学习的过程，这样我们通过SPI机制，可以实现AOP的一写功能。



## 总结

到此，我们从Java SPI开始，了解了Dubbo SPI 的基本概念，并结合了Dubbo中的LoadBalance加深了理解。最后，我们还实践了一下，创建了一个自定义LoadBalance，并集成到Dubbo中。相信通过这里理论和实践的结合，大家对Dubbo的可扩展有更深入的理解。 总结一下，Dubbo SPI有以下的特点:

- 对Dubbo进行扩展，不需要改动Dubbo的源码
- 自定义的Dubbo的扩展点实现，是一个普通的Java类，Dubbo没有引入任何Dubbo特有的元素，对代码侵入性几乎为零。
- 将扩展注册到Dubbo中，只需要在ClassPath中添加配置文件。使用简单。而且不会对现有代码造成影响。符合开闭原则。
- dubbo的扩展机制设计默认值：@SPI("dubbo") 代表默认的spi对象
- Dubbo的扩展机制支持IoC,AoP等高级功能
- Dubbo的扩展机制能很好的支持第三方IoC容器，默认支持Spring Bean，可自己扩展来支持其他容器，比如Google的Guice。
- 切换扩展点的实现，只需要在配置文件中修改具体的实现，不需要改代码。使用方便。

## 参考

文章中间部分和总结，使用的是官方文档中的描述。

[Dubbo可扩展机制实战](http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html)