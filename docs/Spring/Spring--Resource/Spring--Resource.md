##   前言

在前面我们初步简单的分析了一下BeanFactory的体系结构，第一步我们需要从配置文件中读取配置信息，JDK所提供的访问资源的类（如java.net.URL、File等），并不能很好的满足各种底层资源的访问需求，比如缺少从类路径或者Web容器的上下文获取资源的操作类。

Spring 设计了一个Resource接口，它为应用提供了更强的底层资源访问能力。该接口拥有对应不同资源类型的实现类，而且Spring的Resource接口及其实现类可以在脱离Spring框架的情况下使用。

## 资源

###  体系结构

![20180918160314](http://img.blog.ztgreat.cn/document/spring/20180918160314.png)

上图是resource接口的部分实现类

1、`WritableResource`: 可写资源接口

2、`ByteArrayResource`:二进制数组表示的资源

3、`ClassPathResource`:类路径下的资源，资源以相对于类路径的方式表示

4、`FileSystemResource`:文件系统资源，资源以文件系统路径的方式表示，如C://bean.xml

5、`ServletContextResource`:为访问Web容器上下文中的资源而设计的类，负责以相对于Web应用根目录的路径加载资源。它支持以流和URL的方式访问，在WAR解包的情况下，也可以通过File方式访问。该类还可以直接从JAR包中访问资源。

6、`PathResource`:Path封装了java.net.URL、java.nio.file.Path、文件系统资源，它使用户能够访问任何可以通过URL、Path、系统文件路径表示的资源，如文件系统的资源、HTTP资源等。

###  Resource 接口

```
    /**
     * 资源是否存在
     */
	boolean exists();

	 /**
     * 资源是否可读
     */
	default boolean isReadable() {
		return true;
	}

	/**
     * 资源是否被打开了
     */
	default boolean isOpen() {
		return false;
	}

	default boolean isFile() {
		return false;
	}

    /**
     * 返回资源的URL的句柄
     */
	URL getURL() throws IOException;

	/**
     * 返回资源的URI的句柄
     */
	URI getURI() throws IOException;

	
	File getFile() throws IOException;

    /**
     * 通过nio channel 访问
     */
	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	long contentLength() throws IOException;

	long lastModified() throws IOException;

	/**
     * 根据资源的相对路径创建新资源
     */
	Resource createRelative(String relativePath) throws IOException;

	String getFilename();

	//资源描述
	String getDescription();
```

通过定义的接口访问，我们可以查询资源状态，内容，如果需要自定义Resource,可以继承`AbstractResource`覆盖相应的方法即可。

`AbstractResource` 为 Resource 接口的默认实现，它实现了 Resource 接口的大部分的公共实现，鉴于篇幅这里就不展示源码，可以自行查看。

##  资源加载器

资源是有了，但是如何去查找和定位这些资源，则应该是`ResourceLoader`的职责了，`ResourceLoader` 接口是资源查找定位策略的统一抽象，具体的资源查找定位策略则由相应的`ResourceLoader`实现类给出。

###  ResourceLoader

```
public interface ResourceLoader {

	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;

	Resource getResource(String location);

	ClassLoader getClassLoader();

}
```

ResourceLoader 接口提供两个方法：`getResource()`、`getClassLoader()`。

`getResource()`根据所提供资源的路径 location 返回 Resource 实例，但是它不确保该 Resource 一定存在，需要调用 `Resource.exist()`方法判断。该方法支持以下模式的资源加载：

- URL位置资源，如"file:C:/test.dat"
- ClassPath位置资源，如"classpath:test.dat"
- 相对路径资源，如"WEB-INF/test.dat"，此时返回的Resource实例根据实现不同而不同

该方法的主要实现是在其子类 DefaultResourceLoader 中实现，具体过程我们在分析 DefaultResourceLoader 时做详细说明。

`getClassLoader()` 返回 ClassLoader 实例，对于想要获取 ResourceLoader 使用的 ClassLoader 用户来说，可以直接调用该方法来获取.

对于想要获取 ResourceLoader 使用的 ClassLoader 用户来说，可以直接调用 `getClassLoader()` 方法获得。在分析 Resource 时，提到了一个类 ClassPathResource ，这个类是可以根据指定的 ClassLoader 来加载资源的。

作为 Spring 统一的资源加载器，它提供了统一的抽象，具体的实现则由相应的子类来负责实现，下面是ResourceLoader 的部分类结构图：

![20180918175900](http://img.blog.ztgreat.cn/document/spring/20180918175900.png)



####   DefaultResourceLoader

DefaultResourceLoader 是 ResourceLoader 的默认实现，它接收 ClassLoader 作为构造函数的参数或者使用不带参数的构造函数，在使用不带参数的构造函数时，使用的 ClassLoader 为默认的 ClassLoader

```

//自定义资源解析，后面分析
private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<>(4);

public DefaultResourceLoader() {
        this.classLoader = ClassUtils.getDefaultClassLoader();
    }

    public DefaultResourceLoader(@Nullable ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    public void setClassLoader(@Nullable ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    @Override
    @Nullable
    public ClassLoader getClassLoader() {
        return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
    }
```

ResourceLoader 中最主要的方法为 `getResource()`,它根据提供的 location 返回相应的 Resource，而 DefaultResourceLoader 对该方法提供了核心实现：

```
public Resource getResource(String location) {
        Assert.notNull(location, "Location must not be null");

        for (ProtocolResolver protocolResolver : this.protocolResolvers) {
            Resource resource = protocolResolver.resolve(location, this);
            if (resource != null) {
                return resource;
            }
        }

        if (location.startsWith("/")) {
            return getResourceByPath(location);
        }
        else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
            return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
        }
        else {
            try {
                // Try to parse the location as a URL...
                URL url = new URL(location);
                return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            }
            catch (MalformedURLException ex) {
                // No URL -> resolve as resource path.
                return getResourceByPath(location);
            }
        }
    }
```

首先通过 ProtocolResolver 来加载资源，成功返回 Resource，否则执行如下逻辑：

- 若 location 以 / 开头，则调用 `getResourceByPath()`构造 ClassPathContextResource 类型资源并返回。
- 若 location 以 classpath: 开头，则构造 ClassPathResource 类型资源并返回。
- 构造 URL ，尝试通过它进行资源定位，若没有抛出 MalformedURLException 异常，则判断是否为 FileURL , 如果是则构造 FileUrlResource 类型资源，否则构造 UrlResource。若在加载过程中抛出 MalformedURLException 异常，则委派 `getResourceByPath()` 实现资源定位加载。



如果通过URL没有定位到资源，那么将会抛出 `MalformedURLException`异常，此时会通过`getResourceByPath` 来进行资源查找，在DefaultResourceLoader  中getResourceByPath 方法默认实现逻辑是构造`ClassPathResource`类型的资源

```
protected Resource getResourceByPath(String path) {
	return new ClassPathContextResource(path, getClassLoader());
}
```



#####  ProtocolResolver

ProtocolResolver ，用户自定义协议资源解决策略，它允许用户自定义资源加载协议，而不需要继承 ResourceLoader 的子类。在介绍 Resource 时，提到如果要实现自定义 Resource，我们只需要继承 DefaultResource 即可，但是有了 ProtocolResolver 后，我们不需要直接继承 DefaultResourceLoader，改为实现 ProtocolResolver 接口也可以实现自定义的 ResourceLoader。
ProtocolResolver 接口，仅有一个方法 `Resource resolve(String location, ResourceLoader resourceLoader)`，该方法接收两个参数：资源路径location，指定的加载器 ResourceLoader，返回为相应的 Resource 。

在 Spring 中接口并没有实现类，它需要用户自定义，自定义的 Resolver 可以通过调用`DefaultResourceLoader.addProtocolResolver()` 添加，如下：

```
  public void addProtocolResolver(ProtocolResolver resolver) {
        Assert.notNull(resolver, "ProtocolResolver must not be null");
        this.protocolResolvers.add(resolver);
  }
```



####  FileSystemResourceLoader

`FileSystemResourceLoader`继承DefaultResourceLoader，DefaultResourceLoader 在最后getResourceByPath 中通过构造ClassPathContextResource 资源，在FileSystemResourceLoader 中重写了该方法，使之从文件系统加载资源。

```
@Override
protected Resource getResourceByPath(String path) {
  if (path.startsWith("/")) {
  path = path.substring(1);
  }
  return new FileSystemContextResource(path);
}
```



####  ResourcePatternResolver

ResourceLoader 的 `Resource getResource(String location)` 每次只能根据 location 返回一个 Resource，当需要加载多个资源时，只能多次调用 `getResource()` 。

ResourcePatternResolver 是 ResourceLoader 的扩展，它支持根据指定的资源路径匹配模式每次返回多个 Resource 实例。

ResourcePatternResolver 在 ResourceLoader 的基础上增加了 `getResources(String locationPattern)`，以支持根据路径匹配模式返回多个 Resource 实例，同时也新增了一种新的协议前缀 `classpath*:`，该协议前缀由其子类负责实现。

```
public interface ResourcePatternResolver extends ResourceLoader {
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    Resource[] getResources(String locationPattern) throws IOException;
}
```



#####  PathMatchingResourcePatternResolver

PathMatchingResourcePatternResolver 为 ResourcePatternResolver 最常用的子类，它除了支持 ResourceLoader 和 ResourcePatternResolver 新增的 classpath: *前缀外，还支持 Ant 风格的路径匹配模式（类似于 `**/.xml`）。

Ant 风格的资源地址支持3种匹配符：

- ？：匹配文件名中的一个字符
- *：匹配文件名中任意字符
- **：匹配多层路径

至于 PathMatchingResourcePatternResolver 这里就不进入分析了，可以自己去看看，目前我们主要还是关注Spring的体系结构。



##   总结

1、Spring 提供了 Resource 和 ResourceLoader 来统一抽象整个资源及其定位。 将资源的定义和资源的加载区分开了，职责更加单一和清晰。

2、DefaultResource 为 Resource 的默认实现，它对 Resource 接口做了一个统一的实现，子类继承该类后只需要覆盖相应的方法即可，同时对于自定义的 Resource 我们也是继承该类。

3、DefaultResourceLoader 同样也是 ResourceLoader 的默认实现，在自定 ResourceLoader 的时候我们除了可以继承该类外还可以实现 ProtocolResolver 接口来实现自定资源加载协议。

4、DefaultResourceLoader 每次只能返回单一的资源，所以 Spring 针对这个提供了另外一个接口 ResourcePatternResolver ，该接口提供了根据指定的 locationPattern 返回多个资源的策略。其子类 PathMatchingResourcePatternResolver 是一个集大成者的 ResourceLoader ，因为它即实现了 `Resource getResource(String location)` 也实现了 `Resource[] getResources(String locationPattern)`，支持 Ant 风格的路径匹配模式。

5、在DefaultResourceLoader  中 如果定位不到资源，则最后将委派给 `getResourceByPath()` 实现资源定位加载，DefaultResourceLoader  中getResourceByPath 返回一个ClassPathResource 类型的资源,在FileSystemResourceLoader 中 重写了该方法，返回一个 FileSystemResource 类型的资源。

6、ResourcePatternResolver  更多样化的资源查找策略，它支持根据指定的`资源路径匹配模式`每次返回多个 Resource 实例，其主要代表实现类为 PathMatchingResourcePatternResolver。

##  参考

Spring 揭秘

Spring 技术内幕

精通Spring 4.x 企业应用开发实战

[【死磕 Spring】—– IOC 之 Spring 统一资源加载策略](http://cmsblogs.com/?p=2656)