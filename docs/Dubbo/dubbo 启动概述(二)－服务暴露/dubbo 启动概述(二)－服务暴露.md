## 前言

在上篇文章中，了解了ServiceBean在spring 中的加载过程，下面我们再来看看ServiceBean是如何暴露的，这部分参考了官方的文档，但是这里不会分析的很仔细，粒度比较粗，只是简单的介绍一些模块，重在思想，如果一行一行的结束代码，估计也会看得爪瞌睡。

> 本文很大程度上参考了官方文档　[服务导出](https://dubbo.apache.org/zh-cn/docs/source_code_guide/export-service.html)
>
> 并在上面基础上做了增删，很多复杂的地方被我略去了



## 简介

Dubbo 服务导出过程始于 Spring 容器发布刷新事件，Dubbo 在接收到事件后，会立即执行服务导出逻辑。整个逻辑大致可分为三个部分，第一部分是前置工作，主要用于检查参数，组装 URL。第二部分是导出服务，包含导出服务到本地 (JVM)，和导出服务到远程两个过程。第三部分是向注册中心注册服务，用于服务发现。本篇文章将会对这三个部分代码进行简单的分析。

在 Dubbo 中，URL 的作用十分重要。Dubbo 使用 URL 作为配置载体，所有的拓展点都是通过 URL 获取配置。这一点，官方文档中有所说明。

> 采用 URL 作为配置信息的统一格式，所有扩展点都通过传递 URL 携带配置信息。

通过 URL 可让 Dubbo 的各种配置在各个模块之间传递。URL 之于 Dubbo，犹如水之于鱼，非常重要。



更多的可以阅读　[Dubbo 中的 URL 统一模型](https://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-url.html)



## 源码分析

### export

服务导出的入口方法是 ServiceBean 的 onApplicationEvent。onApplicationEvent 是一个事件响应方法，该方法会在收到 Spring 上下文刷新事件后执行服务导出操作。方法代码如下：

```
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 是否有延迟导出 && 是否已导出 && 是不是已被取消导出
    if (isDelay() && !isExported() && !isUnexported()) {
        // 导出服务
        export();
    }
}

public void export() {
    super.export();
    // Publish ServiceBeanExportedEvent
    publishExportEvent();
}
```

这个方法首先会根据条件决定是否导出服务，比如有些服务设置了延时导出，那么此时就不应该在此处导出。还有一些服务已经被导出了，或者当前服务被取消导出了，此时也不能再次导出相关服务,`export` 里面的部分代码暂时我们就不看了，export 内部会调用　`doExportUrls`

### doExportUrls

#### 多协议多注册中心导出服务

Dubbo 允许我们使用不同的协议导出服务，也允许我们向多个注册中心注册服务。Dubbo 在 doExportUrls 方法中对多协议，多注册中心进行了支持。相关代码如下：

```
private void doExportUrls() {
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            // 省略其它代码
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
}
```



上面代码首先是通过 loadRegistries 加载注册中心链接，然后再遍历 ProtocolConfig 集合导出每个服务。并在导出服务的过程中，将服务注册到注册中心.

下面是我本机测试的时候注册中心的URL,以供参考。



```
registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?
application=dubbo-demo-annotation-provider&
dubbo=2.0.2&pid=13949&registry=zookeeper&
timestamp=1575108346349
```



> 20880 是我配置的Dubbo 端口

### doExportUrlsFor1Protocol

 这个方法主要是进行URL的组装，那么下面我们来了解一下 URL 组装的过程。

#### URL组装



```
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    // 如果协议名为空，或空串，则将协议名变量设置为 dubbo
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }

    Map<String, String> map = new HashMap<String, String>();
    // 添加 side、版本、时间戳以及进程号等信息到 map 中
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }

    // 通过反射将对象的字段信息添加到 map 中
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    appendParameters(map, protocolConfig);
    appendParameters(map, this);

    // methods 为 MethodConfig 集合，MethodConfig 中存储了 <dubbo:method> 标签的配置信息
    if (methods != null && !methods.isEmpty()) {
        // 这段代码用于添加 Callback 配置到 map 中，代码太长，待会单独分析
    }

    // 检测 generic 是否为 "true"，并根据检测结果向 map 中添加不同的信息
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        // 为接口生成包裹类 Wrapper，Wrapper 中包含了接口的详细信息，比如接口方法名数组，字段信息等
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        // 添加方法名到 map 中，如果包含多个方法名，则用逗号隔开，比如 method = init,destroy
        if (methods.length == 0) {
            logger.warn("NO method found in service interface ...");
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            // 将逗号作为分隔符连接方法名，并将连接后的字符串放入 map 中
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }

    // 添加 token 到 map 中
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            // 随机生成 token
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
    // 判断协议名是否为 injvm
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }

    // 获取上下文路径
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }

    // 获取 host 和 port
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    // 组装 URL
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
    
    // 省略无关代码
}
```



上面的代码首先是将一些信息，比如版本、时间戳、方法名以及各种配置对象的字段信息放入到 map 中，map 中的内容将作为 URL 的查询字符串。构建好 map 后，紧接着是获取上下文路径、主机名以及端口号等信息。最后将 map 和主机名等数据传给 URL 构造方法创建 URL 对象。需要注意的是，这里出现的 URL 并非 java.net.URL，而是 com.alibaba.dubbo.common.URL。

> 上面代码的注释摘自官方文档

同样的这里我将本机测试的URL贴出来

```
dubbo://192.168.0.57:20880/org.apache.dubbo.demo.DemoService?
anyhost=true&
application=dubbo-demo-annotation-provider&
bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&
bind.ip=192.168.0.57&bind.port=20880&
deprecated=false&dubbo=2.0.2&
dynamic=true&generic=false&
interface=org.apache.dubbo.demo.DemoService&
methods=sayHello,sayOK&
pid=13949&release=&
side=provider&
timestamp=1575108745709
```



> DemoService　有两个方法　sayHello和sayOK
>
> 20880 是我配置的Dubbo 端口



#### 导出 Dubbo 服务

前置工作做完，接下来就可以进行服务导出了。服务导出分为导出到本地 (JVM)，和导出到远程。在深入分析服务导出的源码前，我们先来从宏观层面上看一下服务导出逻辑。如下：



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
            if (registryURLs != null && !registryURLs.isEmpty()) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                    // 加载监视器链接
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        // 将监视器链接作为参数添加到 url 中
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }

                    String proxy = url.getParameter(Constants.PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                    }

                    // 为服务提供类(ref)生成 Invoker
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    // DelegateProviderMetaDataInvoker 用于持有 Invoker 和 ServiceConfig
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    // 导出服务，并生成 Exporter
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                
            // 不存在注册中心，仅导出服务
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```



上面代码根据 url 中的 scope 参数决定服务导出方式，分别如下：

- scope = none，不导出服务
- scope != remote，导出到本地
- scope != local，导出到远程

不管是导出到本地，还是远程。进行服务导出之前，均需要先创建 Invoker，这是一个很重要的步骤,经过上面的过程后，我们的服务就暴露出去了,下面先来简单分析 Invoker 的创建过程

#### Invoker 创建过程

在 Dubbo 中，Invoker 是一个非常重要的模型。在服务提供端，以及服务引用端均会出现 Invoker。Dubbo 官方文档中对 Invoker 进行了说明，这里引用一下。

> Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

既然 Invoker 如此重要，那么我们很有必要搞清楚 Invoker 的用途。Invoker 是由 ProxyFactory 创建而来，Dubbo 默认的 ProxyFactory 实现类是 JavassistProxyFactory。下面我们到 JavassistProxyFactory 代码中，探索 Invoker 的创建过程。如下：　



```
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // 为目标类创建 Wrapper
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    // 创建匿名 Invoker 类对象，并实现 doInvoke 方法。
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            // 调用 Wrapper 的 invokeMethod 方法，invokeMethod 最终会调用目标方法
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```



如上，JavassistProxyFactory 创建了一个继承自 AbstractProxyInvoker 类的匿名对象，并覆写了抽象方法 doInvoke。覆写后的 doInvoke 逻辑比较简单，仅是将调用请求转发给了 Wrapper 类的 invokeMethod 方法。Wrapper 用于“包裹”目标类，Wrapper 是一个抽象类，仅可通过 getWrapper(Class) 方法创建子类。在创建 Wrapper 子类的过程中，子类代码生成逻辑会对 getWrapper 方法传入的 Class 对象进行解析，拿到诸如类方法，类成员变量等信息。以及生成 invokeMethod 方法代码和其他一些方法代码。代码生成完毕后，通过 Javassist 生成 Class 对象，最后再通过反射创建 Wrapper 实例。相关的代码如下：

#### Wrapper

```
 public static Wrapper getWrapper(Class<?> c) { 
    while (ClassGenerator.isDynamicClass(c))
        c = c.getSuperclass();

    if (c == Object.class)
        return OBJECT_WRAPPER;

    // 从缓存中获取 Wrapper 实例
    Wrapper ret = WRAPPER_MAP.get(c);
    if (ret == null) {
        // 缓存未命中，创建 Wrapper
        ret = makeWrapper(c);
        // 写入缓存
        WRAPPER_MAP.put(c, ret);
    }
    return ret;
}
```



Wrapper的创建过程这里就不贴了，含有大量的代码，不容易阅读，下面我直接给大家看看我本机测试半成品Wrapper 中的**invokeMethod**,这个应该就很容易理解了。



```
public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException{ 
    org.apache.dubbo.demo.DemoService w; 
    try{ 
        w = ((org.apache.dubbo.demo.DemoService)$1); 
    }
    catch(Throwable e){ 
        throw new IllegalArgumentException(e); 
    } 
    try{ 
        if( "sayOK".equals( $2 )  &&  $3.length == 1 ) {  
            return ($w)w.sayOK((java.lang.String)$4[0]); 
        } 
        if( "sayHello".equals( $2 )  &&  $3.length == 1 ) {  
            return ($w)w.sayHello((java.lang.String)$4[0]); 
        } 
    } catch(Throwable e) {      
        throw new java.lang.reflect.InvocationTargetException(e);  
    } 
    throw new org.apache.dubbo.common.bytecode.NoSuchMethodException("Not found method \""+$2+"\" in class org.apache.dubbo.demo.DemoService."); 
}
```



可以看到　Wrapper其实是一个代理类，对具体实例里面的方法进行代理和分发。



到这里我们可以大胆猜测了了，当我们的服务类的某个方法被调用的时候，会被代理调用**Invoker**的**doInvoke**方法，而doInvoke方法又会调用Wrapper的invokeMethod方法，最后才是调用某个实例的具体方法。

了解了Invoker,现在我们再来看前面的　`protocol.export`　相关的代码，很显然，这个是在把Invoker 进行导出



#### 导出服务到本地

本节我们来看一下服务导出相关的代码，按照代码执行顺序，本节先来分析导出服务到本地的过程。相关代码如下：

```
private void exportLocal(URL url) {
    // 如果 URL 的协议头等于 injvm，说明已经导出到本地了，无需再次导出
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
            .setProtocol(Constants.LOCAL_PROTOCOL)    // 设置协议头为 injvm
            .setHost(LOCALHOST)
            .setPort(0);
        ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
        // 创建 Invoker，并导出服务，这里的 protocol 会在运行时调用 InjvmProtocol 的 export 方法
        Exporter<?> exporter = protocol.export(
            proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
    }
}
```

exportLocal 方法比较简单，首先根据 URL 协议头决定是否导出服务，若需导出，则创建一个新的 URL 并将协议头、主机名以及端口设置成新的值。然后创建 Invoker，并调用 InjvmProtocol 的 export 方法导出服务。

这里我们关注一下URL:



```
dubbo://192.168.0.57:20880/org.apache.dubbo.demo.DemoService?
anyhost=true&application=dubbo-demo-annotation-provider&
bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&
bind.ip=192.168.0.57&bind.port=20880&deprecated=false&
dubbo=2.0.2&dynamic=true&generic=false&
interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayOK&
pid=18661&release=&side=provider&timestamp=1575118185436
```



下面我们来看一下 InjvmProtocol 的 export 方法都做了哪些事情。

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 创建 InjvmExporter
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```

如上，InjvmProtocol 的 export 方法仅创建了一个 InjvmExporter，无其他逻辑。到此导出服务到本地就分析完了，接下来，我们继续分析导出服务到远程的过程。

#### 导出服务到远程

与导出服务到本地相比，导出服务到远程的过程要复杂不少，其包含了服务导出与服务注册两个过程。这两个过程涉及到了大量的调用，比较复杂。按照代码执行顺序，本节先来分析服务导出逻辑，服务注册逻辑将在下一节进行分析。

这里再贴一下部分代码：



```
// scope != local，导出到远程
if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
    if (registryURLs != null && !registryURLs.isEmpty()) {
        for (URL registryURL : registryURLs) {
            url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
            // 加载监视器链接
            URL monitorUrl = loadMonitor(registryURL);
            if (monitorUrl != null) {
                // 将监视器链接作为参数添加到 url 中
                url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
            }

            String proxy = url.getParameter(Constants.PROXY_KEY);
            if (StringUtils.isNotEmpty(proxy)) {
                registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
            }

            // 为服务提供类(ref)生成 Invoker
            //　这里需要关注一下　URL  *********
            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
            // DelegateProviderMetaDataInvoker 用于持有 Invoker 和 ServiceConfig
            DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

            // 导出服务，并生成 Exporter
            Exporter<?> exporter = protocol.export(wrapperInvoker);
            exporters.add(exporter);
        }

        // 不存在注册中心，仅导出服务
    } else {
        //省略代码
    }
}
```



这里我们需要关注一下Invoker 的URL,下面是我本地测试的URL:



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

其中有部分编码了，但是不影响我们的阅读，到这里我们发现URL发生了变化，我们前面说了URL 的作用十分重要。Dubbo 使用 URL 作为配置载体，所有的拓展点都是通过 URL 获取配置，前面的由以前的dubbo 变成了registry，因此这里会进行服务注册。



下面开始分析，我们把目光移动到 **RegistryProtocol** 的 export 方法上。



```
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        
    // 获取注册中心 URL，以 zookeeper 注册中心为例，得到的示例 URL 如下：
    // zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F172.17.48.52%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider
    URL registryUrl = getRegistryUrl(originInvoker);
        
    // url to export locally
    // 获取服务地址,比如：
    // dubbo://192.168.0.57:20880/org.apache.dubbo.demo.DemoService?
    // anyhost=true&application=dubbo-demo-annotation-provider&
    // bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&
    // bind.ip=192.168.0.57&bind.port=20880&deprecated=false&
    // dubbo=2.0.2&dynamic=true&generic=false&
    // interface=org.apache.dubbo.demo.DemoService&methods=sayHello
    URL providerUrl = getProviderUrl(originInvoker);

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
    //  the same service. Because the subscribed is cached key with the name of the service, it causes the
    //  subscription information to cover.
        
    // 获取订阅 URL，比如：
    // provider://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?
    // category=configurators&check=false&anyhost=true&
    // application=demo-provider&dubbo=2.0.2&generic=false&
    // interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        
    // 创建监听器
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

    providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
    //export invoker
    // 导出服务
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

    // 根据 URL 加载 Registry 实现类，比如 ZookeeperRegistry
    final Registry registry = getRegistry(originInvoker);
        
    // 获取已注册的服务提供者 URL，比如：
   　// dubbo://172.17.48.52:20880/com.alibaba.dubbo.demo.DemoService?
    // anyhost=true&application=demo-provider&dubbo=2.0.2&generic=false&
    // interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello
    final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
        
    // 向服务提供者与消费者注册表中注册服务提供者
    ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
                
    //to judge if we need to delay publish
    //获取 register 参数
    boolean register = providerUrl.getParameter(REGISTER_KEY, true);
    // 根据 register 的值决定是否注册服务
    if (register) {
       // 向注册中心注册服务
       register(registryUrl, registeredProviderUrl);
       providerInvokerWrapper.setReg(true);
    }

    // Deprecated! Subscribe to override rules in 2.6.x or before.
    // 向注册中心进行订阅 override 数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

    exporter.setRegisterUrl(registeredProviderUrl);
    exporter.setSubscribeUrl(overrideSubscribeUrl);
    //Ensure that a new exporter instance is returned every time export
    // 创建并返回 DestroyableExporter
    return new DestroyableExporter<>(exporter);
 }
```



上面代码看起来比较复杂，主要做如下一些操作：

1. 调用 doLocalExport 导出服务
2. 向注册中心注册服务
3. 向注册中心进行订阅 override 数据
4. 创建并返回 DestroyableExporter

在以上操作中，除了创建并返回 DestroyableExporter 没什么难度外，其他几步操作都不是很简单。这其中，导出服务和注册服务是本章要重点分析的逻辑。 



下面先来分析 doLocalExport 方法的逻辑，如下：

```
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
     String key = getCacheKey(originInvoker);
     return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
        // 创建 Invoker 为委托类对象
        Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
        // 调用 protocol 的 export 方法导出服务
        // 返回　ExporterChangeableWrapper
        return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
    });
}
```

接下来，我们把重点放在 Protocol 的 export 方法上。假设运行时协议为 dubbo，此处的 protocol 变量会在运行时加载 DubboProtocol，并调用 **DubboProtocol** 的 export 方法。所以，接下来我们目光转移到 **DubboProtocol** 的 export 方法上，相关分析如下：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();
    // 获取服务标识，理解成服务坐标也行。由服务组名，服务名，服务版本号以及端口组成。比如：
    // demoGroup/com.alibaba.dubbo.demo.DemoService:1.0.1:20880
    String key = serviceKey(url);
    // 创建 DubboExporter
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    // 将 <key, exporter> 键值对放入缓存中
    exporterMap.put(key, exporter);
    // 本地存根相关代码
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            // 省略日志打印代码
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }
    // 启动服务器
    openServer(url);
    // 优化序列化
    optimizeSerialization(url);
    return exporter;
}
```

如上，我们重点关注 DubboExporter 的创建以及 openServer 方法，其他逻辑看不懂也没关系，不影响理解服务导出过程，现有初步印象，肯定需要反复来几次的。

下面简单分析 openServer 方法：



```
private void openServer(URL url) {
    // 获取 host:port，并将其作为服务器实例的 key，用于标识当前的服务器实例
    String key = url.getAddress();
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        // 访问缓存
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            // 创建服务器实例
            serverMap.put(key, createServer(url));
        } else {
            // 服务器已创建，则根据 url 中的配置重置服务器
            server.reset(url);
        }
    }
}
```



一般情况下真正openServer 只有一次，我本机测试的时候第一次的URL如下：



```
dubbo://192.168.0.57:20880/org.apache.dubbo.demo.DemoService?
anyhost=true&application=dubbo-demo-annotation-provider&
bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&
bind.ip=192.168.0.57&bind.port=20880&deprecated=false&dubbo=2.0.2&
dynamic=true&generic=false&
interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayOK&
pid=17608&release=&side=provider&timestamp=1575116449507
```

> 20880 是我配置的Dubbo 端口

这里暂时就不继续往下分析了，现有个大概认识吧，需要详细研究的时候再来看，不然一开始容易晕。

整个过程比较复杂，大家在分析的过程中耐心一些。并且多写 Demo 进行调试，以便能够更好的理解代码逻辑。

本节内容先到这里，接下来分析服务导出的另一块逻辑 — 服务注册。



#### 服务注册

本节我们来分析服务注册过程，服务注册操作对于 Dubbo 来说不是必需的，通过服务直连的方式就可以绕过注册中心。但通常我们不会这么做，直连方式不利于服务治理，仅推荐在测试服务时使用。对于 Dubbo 来说，注册中心虽不是必需，但却是必要的。因此，关于注册中心以及服务注册相关逻辑，我们也需要搞懂。



本节内容以 Zookeeper 注册中心作为分析目标，其他类型注册中心大家可自行分析。下面从服务注册的入口方法开始分析，我们把目光再次移到 **RegistryProtocol** 的 export 方法上。如下：

```
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    
    // ${导出服务}
    
    // 省略其他代码
    
    boolean register = registeredProviderUrl.getParameter("register", true);
    if (register) {
        // 注册服务
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }
    
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    // 订阅 override 数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    // 省略部分代码
}
```

RegistryProtocol 的 export 方法包含了服务导出，注册，以及数据订阅等逻辑。其中服务导出逻辑上一节已经分析过了，本节将分析服务注册逻辑，相关代码如下：

```
public void register(URL registryUrl, URL registedProviderUrl) {
    // 获取 Registry
    Registry registry = registryFactory.getRegistry(registryUrl);
    // 注册服务
    registry.register(registedProviderUrl);
}
```

register 方法包含两步操作，第一步是获取注册中心实例，第二步是向注册中心注册服务。接下来分两节内容对这两步操作进行分析。

#### 创建注册中心

本节内容以 Zookeeper 注册中心为例进行分析。下面先来看一下 getRegistry 方法的源码，这个方法由 AbstractRegistryFactory 实现。如下：

```
public Registry getRegistry(URL url) {
    url = url.setPath(RegistryService.class.getName())
            .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
            .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
    String key = url.toServiceString();
    LOCK.lock();
    try {
        // 访问缓存
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
            return registry;
        }
        
        // 缓存未命中，创建 Registry 实例
        registry = createRegistry(url);
        if (registry == null) {
            throw new IllegalStateException("Can not create registry...");
        }
        
        // 写入缓存
        REGISTRIES.put(key, registry);
        return registry;
    } finally {
        LOCK.unlock();
    }
}
protected abstract Registry createRegistry(URL url);
```

如上，getRegistry 方法先访问缓存，缓存未命中则调用 createRegistry 创建 Registry，然后写入缓存。这里的 createRegistry 是一个模板方法，由具体的子类实现。因此，下面我们到 ZookeeperRegistryFactory 中探究一番。

```
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
    // zookeeperTransporter 由 SPI 在运行时注入，类型为 ZookeeperTransporter$Adaptive
    private ZookeeperTransporter zookeeperTransporter;
    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }
    @Override
    public Registry createRegistry(URL url) {
        // 创建 ZookeeperRegistry
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
}
```

ZookeeperRegistryFactory 的 createRegistry 方法仅包含一句代码，无需解释，继续跟下去。

```
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);
    if (url.isAnyHost()) {
        throw new IllegalStateException("registry address == null");
    }
    
    // 获取组名，默认为 dubbo
    String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
    if (!group.startsWith(Constants.PATH_SEPARATOR)) {
        // group = "/" + group
        group = Constants.PATH_SEPARATOR + group;
    }
    this.root = group;
    // 创建 Zookeeper 客户端，默认为 CuratorZookeeperTransporter
    zkClient = zookeeperTransporter.connect(url);
    // 添加状态监听器
    zkClient.addStateListener(new StateListener() {
        @Override
        public void stateChanged(int state) {
            if (state == RECONNECTED) {
                try {
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
    });
}
```

在上面的代码代码中，我们重点关注 ZookeeperTransporter 的 connect 方法调用，这个方法用于创建 Zookeeper 客户端。创建好 Zookeeper 客户端，意味着注册中心的创建过程就结束了,具体的细节就不往下了。

现在注册中心实例创建好了，接下来要做的事情是向注册中心注册服务，我们继续往下看。

#### 节点创建

以 Zookeeper 为例，**所谓的服务注册，本质上是将服务配置数据写入到 Zookeeper 的某个路径的节点下**。为了让大家有一个直观的了解，下面我们将 Dubbo 的 demo 跑起来，然后通过 Zookeeper 可视化客户端 [ZooInspector](https://github.com/apache/zookeeper/tree/b79af153d0f98a4f3f3516910ed47234d7b3d74e/src/contrib/zooinspector) 查看节点数据。如下：

![img](http://img.blog.ztgreat.cn/document/dubbo/1575114200026.png)

从上图中可以看到 com.alibaba.dubbo.demo.DemoService 这个服务对应的配置信息（存储在 URL 中）最终被注册到了 /dubbo/com.alibaba.dubbo.demo.DemoService/providers/ 节点下。搞懂了服务注册的本质，那么接下来我们就可以去阅读服务注册的代码了。服务注册的接口为 register(URL)，这个方法定义在 FailbackRegistry 抽象类中。代码如下：

```
public void register(URL url) {
    super.register(url);
    failedRegistered.remove(url);
    failedUnregistered.remove(url);
    try {
        // 模板方法，由子类实现
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;
        // 获取 check 参数，若 check = true 将会直接抛出异常
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        if (check || skipFailback) {
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("Failed to register");
        } else {
            logger.error("Failed to register");
        }
        // 记录注册失败的链接
        failedRegistered.add(url);
    }
}
protected abstract void doRegister(URL url);
```

如上，我们重点关注 doRegister 方法调用即可，其他的代码先忽略。doRegister 方法是一个模板方法，因此我们到 FailbackRegistry 子类 ZookeeperRegistry 中进行分析。如下：

```
protected void doRegister(URL url) {
    try {
        // 通过 Zookeeper 客户端创建节点，节点路径由 toUrlPath 方法生成，路径格式如下:
        //   /${group}/${serviceInterface}/providers/${url}
        // 比如
        //   /dubbo/org.apache.dubbo.DemoService/providers/dubbo%3A%2F%2F127.0.0.1......
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register...");
    }
}
```

如上，ZookeeperRegistry 在 doRegister 中调用了 Zookeeper 客户端创建服务节点。节点路径由 toUrlPath 方法生成，该方法逻辑不难理解，就不分析了。接下来分析 create 方法，如下：

```
public void create(String path, boolean ephemeral) {
    if (!ephemeral) {
        // 如果要创建的节点类型非临时节点，那么这里要检测节点是否存在
        if (checkExists(path)) {
            return;
        }
    }
    int i = path.lastIndexOf('/');
    if (i > 0) {
        // 递归创建上一级路径
        create(path.substring(0, i), false);
    }
    
    // 根据 ephemeral 的值创建临时或持久节点
    if (ephemeral) {
        createEphemeral(path);
    } else {
        createPersistent(path);
    }
}
```

上面方法先是通过递归创建当前节点的上一级路径，然后再根据 ephemeral 的值决定创建临时还是持久节点。createEphemeral 和 createPersistent 这两个方法都比较简单，这里简单分析其中的一个。如下：

```
public void createEphemeral(String path) {
    try {
        // 通过 Curator 框架创建节点
        client.create().withMode(CreateMode.EPHEMERAL).forPath(path);
    } catch (NodeExistsException e) {
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

好了，到此关于服务注册的过程就分析完了。整个过程可简单总结为：先创建注册中心实例，之后再通过注册中心实例注册服务。本节先到这，整个过程还是蛮复杂的，需要反复来几遍，最好自己断点更，这一节里面有几个不同的protocol　服务，但是都是通过下面的protocol　导出的。



```
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```



这里就涉及到dubbo 里面的其它概念了，这个再后面来分析，更多的可以阅读　[扩展点加载](https://dubbo.apache.org/zh-cn/docs/dev/SPI.html)

#### 时序图

最后引用官方暴露服务时序图

时序图如下：

![img](http://img.blog.ztgreat.cn/document/dubbo/1575117648949.jpeg)

### 参考

[官方文档](https://dubbo.apache.org/zh-cn/docs/source_code_guide/export-service.html)