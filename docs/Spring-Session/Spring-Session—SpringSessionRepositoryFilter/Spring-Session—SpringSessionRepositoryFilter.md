
##  Tomcat Session
对于session 是一个老生畅谈的话题了，Session管理是JavaEE容器比较重要的一部分，
Tomcat中主要由每个context容器内的一个Manager对象来管理session。对于这个manager对象的实现，可以根据tomcat提供的接口或基类来自己定制，同时，tomcat也提供了标准实现。 

在每个context对象，即web app都具有一个独立的manager对象。通过server.xml可以配置定制化的manager，也可以不配置。不管怎样，在生成context对象时，都会生成一个manager对象。缺省的是StandardManager类，在Tomcat 中，Session 是保存到一个ConcurrentHashMap 中的。

```
/**
 * Minimal implementation of the <b>Manager</b> interface that supports
 * no session persistence or distributable capabilities.  This class may
 * be subclassed to create more sophisticated Manager implementations.
 *
 * @author Craig R. McClanahan
 */
public abstract class ManagerBase extends LifecycleMBeanBase implements Manager {
    /**
     * The set of currently active Sessions for this Manager, keyed by
     * session identifier.
     */
    protected Map<String, Session> sessions = new ConcurrentHashMap<>();
    ...
 }
```

Session 自定义化，可以实现标准servlet的session接口： 

```
javax.servlet.http.HttpSession
```

Tomcat也提供了标准的session实现： 

```
org.apache.catalina.session.StandardSession
```

```
/**
 * Standard implementation of the <b>Session</b> interface.  This object is
 * serializable, so that it can be stored in persistent storage or transferred
 * to a different JVM for distributable session support.
 * <p>
 * <b>IMPLEMENTATION NOTE</b>:  An instance of this class represents both the
 * internal (Session) and application level (HttpSession) view of the session.
 * However, because the class itself is not declared public, Java logic outside
 * of the <code>org.apache.catalina.session</code> package cannot cast an
 * HttpSession view of this instance back to a Session view.
 * <p>
 * <b>IMPLEMENTATION NOTE</b>:  If you add fields to this class, you must
 * make sure that you carry them over in the read/writeObject methods so
 * that this class is properly serialized.
 *
 * @author Craig R. McClanahan
 * @author Sean Legassick
 * @author <a href="mailto:jon@latchkey.com">Jon S. Stevens</a>
 */
public class StandardSession implements HttpSession, Session, Serializable {
    /**
     * Construct a new Session associated with the specified Manager.
     *
     * @param manager The manager with which this Session is associated
     */
    public StandardSession(Manager manager) {

        super();
        this.manager = manager;

        // Initialize access count
        if (ACTIVITY_CHECK) {
            accessCount = new AtomicInteger();
        }
    }
    ...
 }
```


今天的目的不是讨论Tomcat 对session的管理，这里只是做个引子，方便感兴趣的朋友深入了解，今天我们主要看Spring 如果接管 Session的，并且使用Redis作为Session的存储容器。

##  Spring Session

这篇文章并不是深入剖析Spring Session的结构，主要是一起来看看Spring 是如何接管Session的，从中学习一些架构思想。


###  Spring Session 基于XML的配置

**注意：下面的配置是核心配置，而非全部配置**

spring 配置文件：
```
<bean id="redisHttpSessionConfiguration"
    class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration">
	<property name="maxInactiveIntervalInSeconds" value="1800" />
</bean>
```

web.xml:

```
<filter>
	<filter-name>springSessionRepositoryFilter</filter-name>
	<filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
	<filter-name>springSessionRepositoryFilter</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>

```
###  DelegatingFilterProxy

DelegatingFilterProxy 是一个Filter的代理类，DelegatingFilterProxy 继承自 GenericFilterBean，GenericFilterBean是一个抽象类，分别实现了 Filter, BeanNameAware, EnvironmentAware, ServletContextAware, InitializingBean, DisposableBean接口，继承关系如下图：

![这里写图片描述](http://img.blog.ztgreat.cn/document/spring/spring-session/20180802164622646.png)

对于Filter 来说，最重要肯定就是初始化（init）和doFilter 方法了，初始化方法在父类GenericFilterBean 中:

```
/**
 * Standard way of initializing this filter.
 * Map config parameters onto bean properties of this filter, and
 * invoke subclass initialization.
 * @param filterConfig the configuration for this filter
 * @throws ServletException if bean properties are invalid (or required
 * properties are missing), or if subclass initialization fails.
 * @see #initFilterBean
 */
@Override
public final void init(FilterConfig filterConfig) throws ServletException {
	...//省略部分代码

	// Let subclasses do whatever initialization they like.
	initFilterBean();  //注意这里

	if (logger.isDebugEnabled()) {
		logger.debug("Filter '" + filterConfig.getFilterName() + "' configured successfully");
	}
}
```
initFilterBean在子类中实现，也就是说当DelegatingFilterProxy 在执行Filter的 init 方法时，会调用 initFilterBean方法，如下：

```
	@Override
	protected void initFilterBean() throws ServletException {
		synchronized (this.delegateMonitor) {
		    // delegate 为实际的Filter
			if (this.delegate == null) {
				// If no target bean name specified, use filter name.
				if (this.targetBeanName == null) {
				    //这里的targetBeanName  便是我们web.xml 中配置的springSessionRepositoryFilter
					this.targetBeanName = getFilterName();
				}
				// Fetch Spring root application context and initialize the delegate early,
				// if possible. If the root application context will be started after this
				// filter proxy, we'll have to resort to lazy initialization.
				WebApplicationContext wac = findWebApplicationContext();
				if (wac != null) {
				    //初始化Filter
					this.delegate = initDelegate(wac);
				}
			}
		}
	}
```
delegate 为真正的Filter,通过web.xml 中配置的Filter 名字（即:springSessionRepositoryFilter）来获取这个Filter，我们继续往下看initDelegate 这个方法：

```
protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
	Filter delegate = wac.getBean(getTargetBeanName(), Filter.class);
	if (isTargetFilterLifecycle()) {
		delegate.init(getFilterConfig());
	}
	return delegate;
}
```
通过上下文，获取到这个Filter Bean,然后初始化这个Filter。但是我们似乎没有手动配置这个Bean，那么这个切入点就是我们开始在spring 配置文件中声明的 RedisHttpSessionConfiguration 这个bean了。
![这里写图片描述](http://img.blog.ztgreat.cn/document/spring/spring-session/20180802170005910.png)

###  SessionRepositoryFilter
Spring Session 在 RedisHttpSessionConfiguration以及它的父类 SpringHttpSessionConfiguration中 自动生成了许多Bean，这里我只列举了springSessionRepositoryFilter 这个Bean,而这个Bean 是SessionRepositoryFilter 类型的。

```
@Configuration
public class SpringHttpSessionConfiguration {

private CookieHttpSessionStrategy defaultHttpSessionStrategy = new CookieHttpSessionStrategy();

private HttpSessionStrategy httpSessionStrategy = this.defaultHttpSessionStrategy;

@Bean
public <S extends ExpiringSession> SessionRepositoryFilter<? extends ExpiringSession> springSessionRepositoryFilter(
		SessionRepository<S> sessionRepository) {
		//传入 sessionRepository
	SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter<S>(
			sessionRepository);
	sessionRepositoryFilter.setServletContext(this.servletContext);
	if (this.httpSessionStrategy instanceof MultiHttpSessionStrategy) {
		sessionRepositoryFilter.setHttpSessionStrategy(
				(MultiHttpSessionStrategy) this.httpSessionStrategy);
	}
	else {
		sessionRepositoryFilter.setHttpSessionStrategy(this.httpSessionStrategy);
	}
	return sessionRepositoryFilter;
}
```
###  RedisOperationsSessionRepository 
现在我们知道，DelegatingFilterProxy 中真正的Filter是SessionRepositoryFilter，在生成SessionRepositoryFilter 是传入了sessionRepository，通过名字我们大概知道，这个应该是具体操作session的类，而这个类又具体是什么呢？，看下面的代码：

```
@Bean
public RedisOperationsSessionRepository sessionRepository(
		@Qualifier("sessionRedisTemplate") RedisOperations<Object, Object> sessionRedisTemplate,
		ApplicationEventPublisher applicationEventPublisher) {
	RedisOperationsSessionRepository sessionRepository = new RedisOperationsSessionRepository(
			sessionRedisTemplate);
	sessionRepository.setApplicationEventPublisher(applicationEventPublisher);
	sessionRepository
			.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
	if (this.defaultRedisSerializer != null) {
		sessionRepository.setDefaultSerializer(this.defaultRedisSerializer);
	}

	String redisNamespace = getRedisNamespace();
	if (StringUtils.hasText(redisNamespace)) {
		sessionRepository.setRedisKeyNamespace(redisNamespace);
	}

	sessionRepository.setRedisFlushMode(this.redisFlushMode);
	return sessionRepository;
}
```
到这里我们知道，这个sessionRepository 具体就是：RedisOperationsSessionRepository ，这个类就是实际通过redis 操作session的类，这里就不展开了，后面会简单看一下。


好了，现在回到 DelegatingFilterProxy -> initFilterBean 方法，具体的Filter 已经找到了，这个Filter 就是SessionRepositoryFilter，那么这个Filter 又是在什么时候生效的呢?,接下来我们看 DelegatingFilterProxy  的doFilter 方法。

```
DelegatingFilterProxy  -> doFilter:

@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
		throws ServletException, IOException {

	// Lazily initialize the delegate if necessary.
	Filter delegateToUse = this.delegate;
	if (delegateToUse == null) {
		synchronized (this.delegateMonitor) {
			delegateToUse = this.delegate;
			if (delegateToUse == null) {
				WebApplicationContext wac = findWebApplicationContext();
				if (wac == null) {
					throw new IllegalStateException("No WebApplicationContext found: " +
							"no ContextLoaderListener or DispatcherServlet registered?");
				}
				delegateToUse = initDelegate(wac);
			}
			this.delegate = delegateToUse;
		}
	}

	// Let the delegate perform the actual doFilter operation.
	invokeDelegate(delegateToUse, request, response, filterChain);
}
```
注意 invokeDelegate 方法，进去看看：

```
protected void invokeDelegate(
		Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain)
		throws ServletException, IOException {
    //调用 delegate的 doFilter 方法
	delegate.doFilter(request, response, filterChain);
}
```
这里我们大概知道了，DelegatingFilterProxy 实际上是委托给delegate 的，而这里的delegate 是SessionRepositoryFilter，现在我们又回到 SessionRepositoryFilter看看：

![这里写图片描述](http://img.blog.ztgreat.cn/document/spring/spring-session/20180802174807490.png)

SessionRepositoryFilter 没有重写 doFilter 方法，因此查看父类：OncePerRequestFilter，通过名字我们知道，这个Filter 只执行一次

```
OncePerRequestFilter -> doFilter:
/**
 * This {@code doFilter} implementation stores a request attribute for
 * "already filtered", proceeding without filtering again if the attribute is already
 * there.
 */
public final void doFilter(ServletRequest request, ServletResponse response,
		FilterChain filterChain) throws ServletException, IOException {

	if (!(request instanceof HttpServletRequest)
			|| !(response instanceof HttpServletResponse)) {
		throw new ServletException(
				"OncePerRequestFilter just supports HTTP requests");
	}
	HttpServletRequest httpRequest = (HttpServletRequest) request;
	HttpServletResponse httpResponse = (HttpServletResponse) response;
	//查看是否已经过滤了
	boolean hasAlreadyFilteredAttribute = request
			.getAttribute(this.alreadyFilteredAttributeName) != null;

	if (hasAlreadyFilteredAttribute) {

		// Proceed without invoking this filter...
		filterChain.doFilter(request, response);
	}
	else {
		// Do invoke this filter...
		//加入已经过滤标志
		request.setAttribute(this.alreadyFilteredAttributeName, Boolean.TRUE);
		try {
		    //调用 doFilterInternal
			doFilterInternal(httpRequest, httpResponse, filterChain);
		}
		finally {
			// Remove the "already filtered" request attribute for this request.
			request.removeAttribute(this.alreadyFilteredAttributeName);
		}
	}
}
```
在该方法中，会检查该请求是否已经被过滤了，如果过滤过了那么执行下一个Filter，否则放入已过滤标识，然后执行Filter 功能，doFilterInternal 在SessionRepositoryFilter  被重写

```
SessionRepositoryFilter -> doFilterInternal:
@Override
protected void doFilterInternal(HttpServletRequest request,
		HttpServletResponse response, FilterChain filterChain)
				throws ServletException, IOException {
	request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

	SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(
			request, response, this.servletContext);
	SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(
			wrappedRequest, response);
    //包装 request,response
	HttpServletRequest strategyRequest = this.httpSessionStrategy
			.wrapRequest(wrappedRequest, wrappedResponse);
	HttpServletResponse strategyResponse = this.httpSessionStrategy
			.wrapResponse(wrappedRequest, wrappedResponse);

	try {
		filterChain.doFilter(strategyRequest, strategyResponse);
	}
	finally {
	    //更新session
		wrappedRequest.commitSession();
	}
}
```
Spring 接管Tomcat 的session 主要在这里可以看出来，Spring会包装HttpServletRequest ，HttpServletResponse ，当执行了这个Filter后，将包装了 request,response 专递到后面，这样后面所使用的都是spring 包装过的，这样在获取session的接口，就被spring 所控制了，当由服务器返回给用户是，调用commitSession,更新session。

具接下来我们看一下SessionRepositoryRequestWrapper 这个类：

![这里写图片描述](http://img.blog.ztgreat.cn/document/spring/spring-session/2018080218543975.png)


```
private S getSession(String sessionId) {
    //通过sessionRepository 获取session,在redis 中 这里的 sessionRepository 是RedisOperationsSessionRepository 
	S session = SessionRepositoryFilter.this.sessionRepository
			.getSession(sessionId);
	if (session == null) {
		return null;
	}
	session.setLastAccessedTime(System.currentTimeMillis());
	return session;
}

// 重写 父类 getSession 方法
@Override
public HttpSessionWrapper getSession(boolean create) {
	HttpSessionWrapper currentSession = getCurrentSession();
	if (currentSession != null) {
		return currentSession;
	}
	//从当前请求获取sessionId
	String requestedSessionId = getRequestedSessionId();
	if (requestedSessionId != null
			&& getAttribute(INVALID_SESSION_ID_ATTR) == null) {
		S session = getSession(requestedSessionId);
		if (session != null) {
			this.requestedSessionIdValid = true;
			//对Spring session 进行包装(包装成HttpSession)
			currentSession = new HttpSessionWrapper(session, getServletContext());
			currentSession.setNew(false);
			setCurrentSession(currentSession);
			return currentSession;
		}
		else {
			// This is an invalid session id. No need to ask again if
			// request.getSession is invoked for the duration of this request
			setAttribute(INVALID_SESSION_ID_ATTR, "true");
		}
	}
	if (!create) {
		return null;
	}
	S session = SessionRepositoryFilter.this.sessionRepository.createSession();
	session.setLastAccessedTime(System.currentTimeMillis());
	//对Spring session 进行包装(包装成HttpSession)
	currentSession = new HttpSessionWrapper(session, getServletContext());
	setCurrentSession(currentSession);
	return currentSession;
}

```
到这里应该就很清楚了，对HttpServletRequest 进行包装，然后重写对Session操作的接口，内部调用 SessionRepository 的实现类来对session 进行操作，ok,这下终于明白Spring Session 是如何控制session的了，最后进行总结一下：

当我们配置 DelegatingFilterProxy 时，会配置 filter-name:springSessionRepositoryFilter,当我们配置 RedisHttpSessionConfiguration 这个bean时，这个Filter 则由Spring 给我生成，而这个Filter 实际是 ：**SessionRepositoryFilter**。

当有请求到达时，DelegatingFilterProxy  委托给 SessionRepositoryFilter，而它又将HttpServletRequest,HttpServletResponse 进行一定的包装，重写对session操作的接口，然后将包装好的request,response 传递到后续的Filter中，完成了对Session的拦截操作，后续应用操作的Session 都是Spring Session 包装后的Session。


###  参考
[Spring Session 内部实现原理（源码分析）](https://www.jianshu.com/p/1001e9e2cfcf)

[通过Spring Session实现新一代的Session管理](http://www.infoq.com/cn/articles/Next-Generation-Session-Management-with-Spring-Session)

