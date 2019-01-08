在Spring Session 源码分析（1）中，梳理了一下Spring Session 接管Tomcat Session的过程，通过配置DelegatingFilterProxy 和指定过滤器名称，spring 接收到请求后，包装HttpServletRequest ，HttpServletResponse ，然后再传递到后续应用，因此没有特殊的需求，需要把DelegatingFilterProxy  放在第一位，这样才能保证后续应用拿到的都是Spring Session包装后的对象。

# 重温SessionRepositoryFilter

在SessionRepositoryFilter 中会包装HttpServletRequest ，HttpServletResponse，这里我们再来探究一下：

```
@Override
protected void doFilterInternal(HttpServletRequest request,
		HttpServletResponse response, FilterChain filterChain)
				throws ServletException, IOException {
	request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);
	//包装 request,response
	SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(
			request, response, this.servletContext);
	SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(
			wrappedRequest, response);
    //采用的session 策略
	HttpServletRequest strategyRequest = this.httpSessionStrategy
			.wrapRequest(wrappedRequest, wrappedResponse);
	HttpServletResponse strategyResponse = this.httpSessionStrategy
			.wrapResponse(wrappedRequest, wrappedResponse);

	try {
		filterChain.doFilter(strategyRequest, strategyResponse);
	}
	finally {
		wrappedRequest.commitSession();
	}
}
```
这里我们注意到httpSessionStrategy，这个是什么鬼？

# HttpSessionStrategy
从这个名字我们知道，描述的是session 采取的策略，什么策略呢？，其实就是获取和设置sessionId的策略，通常后台获取sessionId 是通过cookie来获取，也把sessionId通过cookie的方式返回给前端，除此之外 也可以在请求头中设置我们的 sessionId，从请求头中获取我们的 sessionId 。因此HttpSessionStrategy 就由此诞生了。

## 类继承层次

![这里写图片描述](http://img.blog.ztgreat.cn/document/spring/spring-session/20180806172627490.png)

上面是简要的类继承层次，并非全部继承关系。


## 重要方法
HttpSessionStrategy 中 只有三个方法：

```
//从request 中获取sessionId
String getRequestedSessionId(HttpServletRequest request);

//当新创建session的时候调用
void onNewSession(Session session, HttpServletRequest request,HttpServletResponse response);
//当session 失效时调用
void onInvalidateSession(HttpServletRequest request, HttpServletResponse response);
```
## HeaderHttpSessionStrategy

HeaderHttpSessionStrategy 实现了 HttpSessionStrategy 接口。 它主要的功能是在 HTTP 请求头中设置我们的 sessionId（从请求头中获取我们的 sessionId ）。**默认的请求头参数为 x-auth-token**。

```
public class HeaderHttpSessionStrategy implements HttpSessionStrategy {
	private String headerName = "x-auth-token";

	public String getRequestedSessionId(HttpServletRequest request) {
		return request.getHeader(this.headerName);
	}

	public void onNewSession(Session session, HttpServletRequest request,
			HttpServletResponse response) {
		response.setHeader(this.headerName, session.getId());
	}

	public void onInvalidateSession(HttpServletRequest request,
			HttpServletResponse response) {
		//置空
		response.setHeader(this.headerName, "");
	}

	/**
	 * The name of the header to obtain the session id from. Default is "x-auth-token".
	 *
	 * @param headerName the name of the header to obtain the session id from.
	 */
	public void setHeaderName(String headerName) {
		Assert.notNull(headerName, "headerName cannot be null");
		this.headerName = headerName;
	}
}
```
代码很简单，就不多说了。

## CookieHttpSessionStrategy

提供通过 cookie 的方式来传递 sessionId。cookie 参数名默认为 SESSION。CookieHttpSessionStrategy 实现了MultiHttpSessionStrategy 接口以及HttpSessionManager 接口。

```
//通过 request cookie 获取session Id
public String getRequestedSessionId(HttpServletRequest request) {
	Map<String, String> sessionIds = getSessionIds(request);
	String sessionAlias = getCurrentSessionAlias(request);
	return sessionIds.get(sessionAlias);
}
```

```
// 遍历cookie 获取cookie
public Map<String, String> getSessionIds(HttpServletRequest request) {
	List<String> cookieValues = this.cookieSerializer.readCookieValues(request);
	String sessionCookieValue = cookieValues.isEmpty() ? ""
			: cookieValues.iterator().next();
	Map<String, String> result = new LinkedHashMap<String, String>();
	StringTokenizer tokens = new StringTokenizer(sessionCookieValue, " ");
	if (tokens.countTokens() == 1) {
		result.put(DEFAULT_ALIAS, tokens.nextToken());
		return result;
	}
	while (tokens.hasMoreTokens()) {
		String alias = tokens.nextToken();
		if (!tokens.hasMoreTokens()) {
			break;
		}
		String id = tokens.nextToken();
		result.put(alias, id);
	}
	return result;
}

```

## Spring Session 数据结构

### RedisSession

HttpSession 底层的Session 实现是RedisSession，RedisSession 被 HttpSessionWrapper 所包装，而HttpSessionWrapper 实现了HttpSession。

![这里写图片描述](http://img.blog.ztgreat.cn/document/spring/spring-session/20180806172732992.png)
```
final class RedisSession implements ExpiringSession {
    private final MapSession cached;
    private Long originalLastAccessTime;
    private Map<String, Object> delta = new HashMap<String, Object>();
    private boolean isNew;
    private String originalPrincipalName;

    /**
     * Creates a new instance ensuring to mark all of the new attributes to be
     * persisted in the next save operation.
     */
    RedisSession() {
        this(new MapSession());
        this.delta.put(CREATION_TIME_ATTR, getCreationTime());
        this.delta.put(MAX_INACTIVE_ATTR, getMaxInactiveIntervalInSeconds());
        this.delta.put(LAST_ACCESSED_ATTR, getLastAccessedTime());
        this.isNew = true;
        flushImmediateIfNecessary();
    }
    public void setNew(boolean isNew) {
        this.isNew = isNew;
    }
    public boolean isExpired() {
        return this.cached.isExpired();
    }
    //从session 取值
    public Object getAttribute(String attributeName) {
        return this.cached.getAttribute(attributeName);
    }
    //设置内容到session 中
    public void setAttribute(String attributeName, Object attributeValue) {
        this.cached.setAttribute(attributeName, attributeValue);
        this.delta.put(getSessionAttrNameKey(attributeName), attributeValue);
        flushImmediateIfNecessary();
    }
    // ... 省略部分内容
}
```

当我们把session 存到redis 后 ，可以在 Redis 中看到如下的数据结构：

```
A) "spring:session:sessions:39feb101-87d4-42c7-ab53-ac6fe0d91925"

B) "spring:session:expirations:1523934840000"

C) "spring:session:sessions:expires:39feb101-87d4-42c7-ab53-ac6fe0d91925"
```
发现，spring session 会保存三种key，为了统一叙述，在此将他们进行编号，后续简称为 A 类型键，B 类型键，C 类型键，这个又是什么情况？？,这个可以参考下面的文章，通过渐进式的描述，很容易启发自身的思考，可以仔细看看哟

[从Spring-Session源码看Session机制的实现细节](https://www.cnkirito.moe/spring-session-4/)


# 总结
通过对HttpSpring 源码的学习，更加了解Spring Session的工作原理，同时也学到一些架构上的设计，还是值得慢慢回味的，Redis 不保证能及时删除过期的键值，因此在Spring Session 中，引入了定时器和很多辅助的键值对来辅助过期键的删除，当然这个也在一定程度上增加了代码的复杂性，同时也增加了Redis的内存消耗，其思想还是值得学习的。