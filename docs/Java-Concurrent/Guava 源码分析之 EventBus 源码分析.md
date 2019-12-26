# Guava EventBus 源码分析



## 前言

EventBus是Guava的事件处理机制，是设计模式中的观察者模式（生产/消费者编程模型）的优雅实现。

EventBus是一个非常优雅和简单解决方案，我们不用创建复杂的类和接口层次结构。

传统上，Java的**进程内事件分发**都是通过发布者和订阅者之间的显式注册实现的。设计EventBus就是为了取代这种显示注册方式，使组件间有了更好的解耦。

EventBus不是通用型的发布-订阅实现，不适用于进程间通信(不同主机)，对于进程间通信 更多的是采用MQ进行解耦，这里我们不说MQ的优势了，先看看进程内的事件分发EventBus 如何做的。

## EventBus 使用

EventBus的使用是非常简单的，首先你要添加`Guava`的依赖到自己的项目中。这里我们通过一个订单支付的例子来说明`EveentBus`是如何使用的。

```
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>27.1-jre</version>
</dependency>
```

在 guava 中，EventBus  是同步类型的，如果要使用异步类型的EventBus ，则需要使用`AsyncEventBus `，AsyncEventBus  继承 EventBus，构造参数调整了，这里我们直接分析 EventBus  就可以了。

### 1.0 定义EventBus

这里我们通过简单的单例模式创建一个EventBus 实例

```
public class EventBusCenter {

    private static EventBus eventBus = new EventBus();
    private EventBusCenter() {
    }
    public static EventBus getInstance() {
        return eventBus;
    }
    //注册 监听者
    public static void register(Object listener) {
        eventBus.register(listener);
    }
    // 取消 监听者
    public static void unregister(Object listener) {
        eventBus.unregister(listener);
    }
    // 发表事件
    public static void post(Object event) {
        eventBus.post(event);
    }

}
```

###  1.1 定义支付成功事件

```
/**
 * 支付成功事件
 */
public class OrderPaySuccessEvent {

    private String orderNumber;

    private String productId;

    public String getOrderNumber() {
        return orderNumber;
    }

    public String getProductId() {
        return productId;
    }

    public static final class OrderPaySuccessEventBuilder {
        private String orderNumber;
        private String productId;

        private OrderPaySuccessEventBuilder() {
        }

        public static OrderPaySuccessEventBuilder anOrderPaySuccessEvent() {
            return new OrderPaySuccessEventBuilder();
        }

        public OrderPaySuccessEventBuilder orderNumber(String orderNumber) {
            this.orderNumber = orderNumber;
            return this;
        }

        public OrderPaySuccessEventBuilder productId(String productId) {
            this.productId = productId;
            return this;
        }

        public OrderPaySuccessEvent build() {
            OrderPaySuccessEvent orderPaySuccessEvent = new OrderPaySuccessEvent();
            orderPaySuccessEvent.orderNumber = this.orderNumber;
            orderPaySuccessEvent.productId = this.productId;
            return orderPaySuccessEvent;
        }
    }

    @Override
    public String toString() {
        return "OrderPaySuccessEvent{" +
                "orderNumber='" + orderNumber + '\'' +
                ", productId='" + productId + '\'' +
                '}';
    }
}
```

### 1.2 定义支付超时事件

```
/**
 * 支付超时事件
 */
public class OrderPayTimeOutEvent {

    private String orderNumber;

    private String productId;


    public String getOrderNumber() {
        return orderNumber;
    }

    public String getProductId() {
        return productId;
    }

    public static final class OrderPayTimeOutEventBuilder {
        private String orderNumber;
        private String productId;

        private OrderPayTimeOutEventBuilder() {
        }

        public static OrderPayTimeOutEventBuilder anOrderPayTimeOutEvent() {
            return new OrderPayTimeOutEventBuilder();
        }

        public OrderPayTimeOutEventBuilder orderNumber(String orderNumber) {
            this.orderNumber = orderNumber;
            return this;
        }

        public OrderPayTimeOutEventBuilder productId(String productId) {
            this.productId = productId;
            return this;
        }

        public OrderPayTimeOutEvent build() {
            OrderPayTimeOutEvent orderPayTimeOutEvent = new OrderPayTimeOutEvent();
            orderPayTimeOutEvent.orderNumber = this.orderNumber;
            orderPayTimeOutEvent.productId = this.productId;
            return orderPayTimeOutEvent;
        }
    }

    @Override
    public String toString() {
        return "OrderPayTimeOutEvent{" +
                "orderNumber='" + orderNumber + '\'' +
                ", productId='" + productId + '\'' +
                '}';
    }
}
```



### 1.3 定义支付成功监听者

```
/**
 * 订单支付成功监听者
 */
public class OrderPaySuccessListener {

    /**
     * 只有通过@Subscribe注解的方法才会被注册进EventBus
     * 而且方法有且只能有1个参数
     * @param event
     */
    @Subscribe
    public void orderSubscribe(OrderPaySuccessEvent event) {
        // 订单中心处理其它业务
        System.out.println("处理订单中心的支付成功事件: " + event);
    }

    @Subscribe
    public void memberSubscribe(OrderPaySuccessEvent event) {
        // 这里可以通过rpc 调用会员中心业务，(不使用MQ)
        System.out.println("处理会员中心的支付成功事件: " + event);
    }

}
```

### 1.4 定义支付超时监听者

```
public class OrderPayTimeOutListener {

    /**
     * 只有通过@Subscribe注解的方法才会被注册进EventBus
     * 而且方法有且只能有1个参数
     * @param event
     */
    @Subscribe
    public void orderSubscribe(OrderPayTimeOutEvent event) {
        // 订单中心处理其它业务
        System.out.println("处理订单中心的支付超时事件: " + event);
    }

    @Subscribe
    public void memberSubscribe(OrderPayTimeOutEvent event) {
        // 这里可以通过rpc 调用会员中心业务
        System.out.println("处理会员中心的支付超时事件: " + event);
    }

}
```



### 1.5 场景测试

```
public class TestEventBus {

    public static void main(String[] args) throws InterruptedException {

        OrderPaySuccessListener successListener = new OrderPaySuccessListener();
        OrderPayTimeOutListener timeOutListener = new OrderPayTimeOutListener();

        EventBusCenter.register(successListener);
        EventBusCenter.register(timeOutListener);

        System.out.println("============   支付成功事件  ====================");

        //支付成功事件
        OrderPaySuccessEvent.OrderPaySuccessEventBuilder successEventBuilder = OrderPaySuccessEvent.OrderPaySuccessEventBuilder.anOrderPaySuccessEvent();
        successEventBuilder.orderNumber("124").productId("3456");

        EventBusCenter.post(successEventBuilder.build());

        System.out.println("============   支付超时事件  ====================");


        //支付超时事件
        OrderPayTimeOutEvent.OrderPayTimeOutEventBuilder timeOutEventBuilder= OrderPayTimeOutEvent.OrderPayTimeOutEventBuilder.anOrderPayTimeOutEvent();
        timeOutEventBuilder.orderNumber("124").productId("3456");

        EventBusCenter.post(timeOutEventBuilder.build());

    }

}
```

执行 结果：

```
============   支付成功事件  ====================
处理订单中心的支付成功事件: OrderPaySuccessEvent{orderNumber='124', productId='3456'}
处理会员中心的支付成功事件: OrderPaySuccessEvent{orderNumber='124', productId='3456'}
============   支付超时事件  ====================
处理订单中心的支付超时事件: OrderPayTimeOutEvent{orderNumber='124', productId='3456'}
处理会员中心的支付超时事件: OrderPayTimeOutEvent{orderNumber='124', productId='3456'}

Process finished with exit code 0

```

首先，这里我们封装了两个事件对象`Event`，两个监听者对象`EventListener`，并将上述监听者实例注册到EvenBus 中。

然后 我们使用`EventBus`实例发布事件`Event`（支付成功，和支付超时）。然后，以上注册的监听者中的使用`@Subscribe`注解声明并且只有一个`Event`类型的参数的方法将会在触发事件的时候被触发。

### 1.6 小结

对于普通的 `观察者模式` 在观察者模式中，每个观察者都要实现一个接口，发布事件的时候，我们只要调用接口的方法就行，但是`EventBus`把这个限制设定得更加宽泛，也就是监听者无需实现任何接口，只要方法使用了注解`@Subscribe`并且参数匹配即可,这样不同的方法相当于不同的观察者，这样根据的灵活，而且 同一个监听者可以监听多种类型的事件，也可以在多次监听同一个事件。

## EventBus源码分析

### 2.1 分析之前

好了，通过上面的例子，我们了解了EventBus最基本的使用方法。下面我们来分析一下在`Guava`中是如何为我们实现这个API的。

假如要我们去设计这样一个API，最简单的方式就是在观察者模式上进行拓展：每次调用`EventBus.post()`方法的时候，会对所有的观察者对象进行遍历，然后获取它们全部的方法，判断该方法是否使用了`@Subscribe`并且方法的参数类型是否与`post()`方法发布的事件类型一致，如果一致的话，那么我们就使用反射来触发这个方法。

从上面的分析中可以看出，这里面不仅要对所有的监听者进行遍历，还要对它们的方法进行遍历，找到了匹配的方法之后又要使用反射来触发这个方法。首先，当注册的**监听者数量比较多**的时候，链式调用的效率就不高；然后我们又要使用反射来触发匹配的方法，这样效率肯定又低了一些，后面我们将在源码分析中来看看 `Guava`的`EventBus`中是如何解决这个问题的



### 2.2 Subscribe 注解

```
@Subscribe
public void orderSubscribe(OrderPayTimeOutEvent event) {
    // 订单中心处理其它业务
    System.out.println("处理订单中心的支付超时事件: " + event);
}
```

在方法上标注 @Subscribe 那么表明这个方法是一个观察者

#### AllowConcurrentEvents 注解

```
@Subscribe
@AllowConcurrentEvents
public void orderSubscribe(OrderPayTimeOutEvent event) {
    // 订单中心处理其它业务
    System.out.println("处理订单中心的支付超时事件: " + event);
}
```

在方法上如果没有标注 @AllowConcurrentEvents注解，会被包装成**SynchronizedEventSubscriber**，即**同步订阅者对象**,这个我们在后面再来说。

### 2.3 初始化EvenBus

首先，当我们使用`new`初始化一个EventBus的时候，实际都会调用到下面的这个方法：

```

private final SubscriberRegistry subscribers = new SubscriberRegistry(this);

EventBus(
      String identifier,
      Executor executor,
      Dispatcher dispatcher,
      SubscriberExceptionHandler exceptionHandler) {
    this.identifier = checkNotNull(identifier);
    this.executor = checkNotNull(executor);
    this.dispatcher = checkNotNull(dispatcher);
    this.exceptionHandler = checkNotNull(exceptionHandler);
}
```

`identifier`是 EventBus的 标识符(名称)； 

 `executor`是事件分发过程中使用到的线程池，可以自己实现；

`dispatcher`是Dispatcher类型的子类，用来在发布事件的时候分发消息给监听者，它有几个默认的实现，分别针对不同的分发方式；

  `exceptionHandler`是SubscriberExceptionHandler类型的，它用来处理异常信息，在默认的EventBus实现中，会在出现异常的时候打印出log，当然我们也可以定义自己的异常处理策咯。

`subscribers`是SubscriberRegistry类型的，从名字就可以猜到  所有的`观察者信息`都维护在该实例中；

### 2.4 数据结构

为 了后面好理解 我们先来看一下 `subscribers`的数据结构

```
/**
 * All registered subscribers, indexed by event type.
 *
 * <p>The {@link CopyOnWriteArraySet} values make it easy and relatively lightweight to get an
 * immutable snapshot of all current subscribers to an event without any locking.
 */
private final ConcurrentMap<Class<?>, CopyOnWriteArraySet<Subscriber>> subscribers =
    Maps.newConcurrentMap();
```

 从注释 我们可以知道，这个 subscribers 存放的是 `事件类->观察者集合`  的一个映射,当发布事件的时候，只要根据事件类型 就可以很快的找到所有的观察者。 

这里的Subscriber列表使用的是Java中的CopyOnWriteArraySet集合， 它内部 使用了CopyOnWriteArrayList，并对其进行了封装，也就是在基本的集合上面增加了去重的操作。这是一种适用于**读多写少**场景的集合，在读取数据的时候不会加锁， 写入数据的时候进行加锁，并且会进行一次数组拷贝。

```
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    private static final long serialVersionUID = 5457747651344034263L;

    private final CopyOnWriteArrayList<E> al;

    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
    ...
}
```

### 2.5 register

我们还是从监听者注册 接口 开始分析

```
/**
 * Registers all subscriber methods on {@code object} to receive events.
 *
 * @param object object whose subscriber methods should be registered.
 */
public void register(Object object) {
  subscribers.register(object);
}
```

从注释 我们可以知道，这个是注册监听者中的所有 被`@subscriber`标记的方法(观察者)

```

/** Registers all subscriber methods on the given listener object. */
void register(Object listener) {
  // 获取指定监听者对应的全部观察者集合
  Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

  //遍历 事件类型->观察者列表  集合
  for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
    //事件类型
    Class<?> eventType = entry.getKey();
    //观察者集合
    Collection<Subscriber> eventMethodsInListener = entry.getValue();

    CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);

    if (eventSubscribers == null) {
      CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<>();
      eventSubscribers =
          MoreObjects.firstNonNull(subscribers.putIfAbsent(eventType, newSet), newSet);
    }
    // 添加 观察者 到对于的事件集合中
    eventSubscribers.addAll(eventMethodsInListener);
  }
}

/**
 * Returns all subscribers for the given listener grouped by the type of event they subscribe to.
 */
private Multimap<Class<?>, Subscriber> findAllSubscribers(Object listener) {
    // 创建一个哈希表
    Multimap<Class<?>, Subscriber> methodsInListener = HashMultimap.create();
    // 获取监听者的类型
    Class<?> clazz = listener.getClass();
    // 获取上述监听者的全部监听方法
    // 遍历上述方法，并且根据方法和类型参数创建观察者并将其插入到映射表中
    for (Method method : getAnnotatedMethods(clazz)) {
      Class<?>[] parameterTypes = method.getParameterTypes();
      // 事件类型
      Class<?> eventType = parameterTypes[0];
      // 保存  事件类型->观察者信息
      methodsInListener.put(eventType, Subscriber.create(bus, listener, method));
    }
    return methodsInListener;
}

```

#####  findAllSubscribers

首先是`findAllSubscribers()`方法，它用来获取指定监听者对应的全部观察者集合。

##### Multimap

这里注意一下`Multimap`数据结构，它是Guava中提供的集合结构，与普通的哈希表不同的地方在于，它可以完成一对多操作。这里用来存储事件类型到观察者的一对多映射。 

```
@Override
public boolean put(@NullableDecl K key, @NullableDecl V value) {
  //value 里面是存放的集合
  Collection<V> collection = map.get(key);
  if (collection == null) {
    collection = createCollection(key);
    if (collection.add(value)) {
      totalSize++;
      map.put(key, collection);
      return true;
    } else {
      throw new AssertionError("New Collection violated the Collection spec");
    }
  } else if (collection.add(value)) {
    totalSize++;
    return true;
  } else {
    return false;
  }
}
```

##### getAnnotatedMethods

在 `findAllSubscribers()`方法中 `getAnnotatedMethods()`方法会尝试从`subscriberMethodsCache`中获取所有的注册监听的方法（即使用了注解并且只有一个参数），下面是这个方法的定义：

```
private static ImmutableList<Method> getAnnotatedMethods(Class<?> clazz) {
    return (ImmutableList)subscriberMethodsCache.getUnchecked(clazz);
}

```

这里的`subscriberMethodsCache`的定义是：

```
private static final LoadingCache<Class<?>, ImmutableList<Method>> subscriberMethodsCache = CacheBuilder.newBuilder().weakKeys().build(new CacheLoader<Class<?>, ImmutableList<Method>>() {
    public ImmutableList<Method> load(Class<?> concreteClass) throws Exception {
        return SubscriberRegistry.getAnnotatedMethodsNotCached(concreteClass);
    }
});

```

这里的作用机制是：当使用`subscriberMethodsCache.getUnchecked(clazz)`获取指定监听者中的方法的时候会先尝试从缓存中进行获取，如果缓存中不存在就会执行load 方法， 调用SubscriberRegistry中的`getAnnotatedMethodsNotCached()`方法获取这些监听方法。这里我们省去该方法的定义，具体可以看下源码中的定于，其实就是使用反射并完成一些校验，并不复杂。

`Subscriber.create` 方法：

```
/** Creates a {@code Subscriber} for {@code method} on {@code listener}. */
static Subscriber create(EventBus bus, Object listener, Method method) {
  return isDeclaredThreadSafe(method)
      ? new Subscriber(bus, listener, method)
      : new SynchronizedSubscriber(bus, listener, method);
}
```

```
static final class SynchronizedSubscriber extends Subscriber {

  private SynchronizedSubscriber(EventBus bus, Object target, Method method) {
    super(bus, target, method);
  }

  @Override
  void invokeSubscriberMethod(Object event) throws InvocationTargetException {
    synchronized (this) {
      super.invokeSubscriberMethod(event);
    }
  }
}
```

`@Subscribe`注解的所有方法，然后对针对每一个listener对象和method方法，标识唯一一个订阅者。

找到唯一识别的观察者后，会对该观察者进行包装wrap，包装成一个EventSubscriber对象，对于没有`@AllowConcurrentEvents`注解的方法，会被包装成**SynchronizedEventSubscriber**，即**同步订阅者对象**。

##### 小结

当注册监听者的时候，首先会拿到该监听者的类型，然后从缓存中尝试获取该监听者对应的所有监听方法，如果没有的话就遍历该类的方法进行获取，并添加到缓存中； 然后，会遍历上述拿到的方法集合，根据事件的类型（从方法参数得知）和监听者等信息创建一个观察者，并将`事件类型-观察者`键值对插入到一个一对多映射表中并返回，最后 根据 事件类型的不同 将观察者者注册到 subscribers 中。 

### 2.6 unregister

`unregister()`  实际上就是把注册的观察者 从集合中移除掉，大体上和 register 差不多，这里我们就不多分析了。



### 2.7 事件触发POST

看下当调用`EventBus.post()`方法的时候的逻辑：

```
public void post(Object event) {
    // 调用SubscriberRegistry的getSubscribers方法获取该事件对应的全部观察者
    Iterator<Subscriber> eventSubscribers = this.subscribers.getSubscribers(event);
    if (eventSubscribers.hasNext()) {
        // 使用Dispatcher对事件进行分发
        this.dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {
        this.post(new DeadEvent(this, event));
    }
}

```

从上面的代码可以看出，实际上当调用`EventBus.post()`方法的时候回先用SubscriberRegistry的getSubscribers方法获取该事件对应的全部观察者，所以我们需要先看下这个逻辑。 以下是该方法的定义：

```
/**
 * Gets an iterator representing an immutable snapshot of all subscribers to the given event at
 * the time this method is called.
 */
Iterator<Subscriber> getSubscribers(Object event) {
  // 获取事件类型的所有父类型和自身构成的集合
  ImmutableSet<Class<?>> eventTypes = flattenHierarchy(event.getClass());

  List<Iterator<Subscriber>> subscriberIterators =
      Lists.newArrayListWithCapacity(eventTypes.size());
  // 遍历上述事件类型，并从subscribers中获取所有的观察者列表
  for (Class<?> eventType : eventTypes) {
    CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);
    if (eventSubscribers != null) {
      // eager no-copy snapshot
      subscriberIterators.add(eventSubscribers.iterator());
    }
  }

  return Iterators.concat(subscriberIterators.iterator());
}
```



这里注意下 `flattenHierarchy`方法，它用来获取当前事件的所有的父类包含自身的类型构成的集合，也就是说，加入我们触发了一个Interger或者String 类型的事件，Object等类型的监听方法都能接收到这个事件并触发。这里的逻辑很简单，就是根据事件的类型，找到它及其所有的父类的类型对应的观察者并返回。

上面我们看到 在获取到事件的观察者后，会通过`Dispatcher`进行分发，下面我们就来简单的看看 Dispatcher。

### 2.8 Dispatcher

接下来我们看真正的分发事件的逻辑是什么样的。

从`EventBus.post()`方法可以看出，当我们使用Dispatcher进行事件分发的时候，需要将当前的事件和所有的观察者作为参数传入到方法中。然后，在方法的内部进行分发操作。最终某个监听者的监听方法是使用反射进行触发的，这部分逻辑在`Subscriber`内部，而Dispatcher是事件分发的方式的策略接口。EventBus中提供了3个默认的Dispatcher实现，分别用于不同场景的事件分发:

`ImmediateDispatcher`：直接在当前线程中遍历所有的观察者并进行事件分发；

```
/**
 * Returns a dispatcher that dispatches events to subscribers immediately as they're posted
 * without using an intermediate queue to change the dispatch order. This is effectively a
 * depth-first dispatch order, vs. breadth-first when using a queue.
 */
static Dispatcher immediate() {
  return ImmediateDispatcher.INSTANCE;
}
```

`LegacyAsyncDispatcher`：使用**全局队列**，先不断往全局的队列中塞入封装的观察者对象，在不断从队列中取出观察者对象进行事件分发；

```
/**
 * Returns a dispatcher that queues events that are posted in a single global queue. This behavior
 * matches the original behavior of AsyncEventBus exactly, but is otherwise not especially useful.
 * For async dispatch, an {@linkplain #immediate() immediate} dispatcher should generally be
 * preferable.
 */
static Dispatcher legacyAsync() {
  return new LegacyAsyncDispatcher();
}
```

`PerThreadQueuedDispatcher`：使用**线程相关队列**，当`dispatch()`方法被调用的时候，会先获取当前线程的观察者队列，并将传入的观察者列表传入到该队列中；判断当前线程是否正在进行分发操作，如果没有在进行分发操作，就通过遍历上述队列进行事件分发。

```
/**
 * Returns a dispatcher that queues events that are posted reentrantly on a thread that is already
 * dispatching an event, guaranteeing that all events posted on a single thread are dispatched to
 * all subscribers in the order they are posted.
 *
 * <p>When all subscribers are dispatched to using a <i>direct</i> executor (which dispatches on
 * the same thread that posts the event), this yields a breadth-first dispatch order on each
 * thread. That is, all subscribers to a single event A will be called before any subscribers to
 * any events B and C that are posted to the event bus by the subscribers to A.
 */
static Dispatcher perThreadDispatchQueue() {
  return new PerThreadQueuedDispatcher();
}
```

上述三个分发器内部最终都会调用Subscriber的`dispatchEvent()`方法进行事件分发：

```
final void dispatchEvent(final Object event) {
    // 使用指定的执行器执行任务
    this.executor.execute(new Runnable() {
        public void run() {
            try {
                // 使用反射触发监听方法
                Subscriber.this.invokeSubscriberMethod(event);
            } catch (InvocationTargetException var2) {
                // 使用EventBus内部的SubscriberExceptionHandler处理异常
                Subscriber.this.bus.handleSubscriberException(var2.getCause(), Subscriber.this.context(event));
            }
        }
    });
}
```

上述方法中的`executor`是执行器，它是通过`EventBus`获取到的；处理异常的SubscriberExceptionHandler类型也是通过`EventBus`获取到的。（原来EventBus中的构造方法中的字段是在这里用到的！）至于反射触发方法调用并没有太复杂的逻辑。

下面说一下三种Dispatcher，都不难，看源码都可以明白的。

#### ImmediateDispatcher

```
/** Implementation of {@link #immediate()}. */
private static final class ImmediateDispatcher extends Dispatcher {
  private static final ImmediateDispatcher INSTANCE = new ImmediateDispatcher();

  @Override
  void dispatch(Object event, Iterator<Subscriber> subscribers) {
    checkNotNull(event);
    while (subscribers.hasNext()) {
      subscribers.next().dispatchEvent(event);
    }
  }
}
```

这个很简单 直接在当前线程中遍历所有的观察者并进行事件分发；

#### LegacyAsyncDispatcher

```
/** Implementation of a {@link #legacyAsync()} dispatcher. */
private static final class LegacyAsyncDispatcher extends Dispatcher {


  /** 全局队列 */
  private final ConcurrentLinkedQueue<EventWithSubscriber> queue =
      Queues.newConcurrentLinkedQueue();

  @Override
  void dispatch(Object event, Iterator<Subscriber> subscribers) {
    checkNotNull(event);
    // 先加队列
    while (subscribers.hasNext()) {
      queue.add(new EventWithSubscriber(event, subscribers.next()));
    }

    EventWithSubscriber e;
    //再分发事件
    while ((e = queue.poll()) != null) {
      e.subscriber.dispatchEvent(e.event);
    }
  }
  ...
}
```

先将任务加入队列，然后再从队列中取任务进行分发，先来的任务会先被执行。所有的事件任务全部放在一个队列中,更加注重 **事件分发的"深度"**。

#### PerThreadQueuedDispatcher

```
/** Implementation of a {@link #perThreadDispatchQueue()} dispatcher. */
private static final class PerThreadQueuedDispatcher extends Dispatcher {

  // This dispatcher matches the original dispatch behavior of EventBus.

  /** 线程相关的队列 */
  private final ThreadLocal<Queue<Event>> queue =
      new ThreadLocal<Queue<Event>>() {
        @Override
        protected Queue<Event> initialValue() {
          return Queues.newArrayDeque();
        }
      };

  /** 标识当前线程状态（是否 dispatch 中） */
  private final ThreadLocal<Boolean> dispatching =
      new ThreadLocal<Boolean>() {
        @Override
        protected Boolean initialValue() {
          return false;
        }
      };

  @Override
  void dispatch(Object event, Iterator<Subscriber> subscribers) {
    checkNotNull(event);
    checkNotNull(subscribers);
    Queue<Event> queueForThread = queue.get();
    // 将需要分发的事件 放入队列
    queueForThread.offer(new Event(event, subscribers));
     
    if (!dispatching.get()) {
      //当前线程没有分发事件中
      dispatching.set(true);
      try {
        Event nextEvent;
        while ((nextEvent = queueForThread.poll()) != null) {
          //分发事件
          while (nextEvent.subscribers.hasNext()) {
            nextEvent.subscribers.next().dispatchEvent(nextEvent.event);
          }
        }
      } finally {
        //移除标识
        dispatching.remove();
        //移除队列，释放内存，不然会有内存泄漏
        queue.remove();
      }
    }
  }
  ...
}
```

事件任务放到不同的线程队列中，没有像 LegacyAsyncDispatcher 那样使用全局队列，更加注重 **事件分发的"广度".**

### 2.9 Executor

在前面2.3 初始化EvenBus的时候，我们知道 EvenBus 初始化的 Executor 默认为`MoreExecutors.directExecutor()`，我们看看 DirectExecutor实现

```
enum DirectExecutor implements Executor {
  INSTANCE;

  @Override
  public void execute(Runnable command) {
    command.run();
  }

  @Override
  public String toString() {
    return "MoreExecutors.directExecutor()";
  }
}
```

我们发现 execute 方法中直接使用的 `command.run()`，并没有使用线程池，因此默认是串行执行的，`Guava`也提供其它的`ExecutorService`,这个可以自己去看看，当然 我们可以自己定义Executor,利用线程池来执行任务。

## AsyncEventBus

EvenBus 是同步类型的，如果要使用异步类型的EventBus，那么可以了解一下AsyncEventBus，就是构造参数不一样了而已。

```
public class AsyncEventBus extends EventBus {

  public AsyncEventBus(String identifier, Executor executor) {
    super(identifier, executor, Dispatcher.legacyAsync(), LoggingHandler.INSTANCE);
  }

  public AsyncEventBus(Executor executor, SubscriberExceptionHandler subscriberExceptionHandler) {
    super("default", executor, Dispatcher.legacyAsync(), subscriberExceptionHandler);
  }

  public AsyncEventBus(Executor executor) {
    super("default", executor, Dispatcher.legacyAsync(), LoggingHandler.INSTANCE);
  }
}
```

## 总结

至此，我们已经简单的完成了EventBus的源码分析。简单总结一下：

通过 `@Subscribe` 注解可以将一个方法变成观察者，`@AllowConcurrentEvents` 允许观察者方法并行执行，如果不加这个注解，该观察者方法会被加锁 串行执行。

每次使用EventBus注册和取消注册监听者的时候，都会先从缓存中进行获取，不是每一次都会用到反射的，这可以提升获取的效率，当使用反射触发方法的调用貌似是不可避免的了。

当post事件的时候，会根据dispatcher的实现来进行不同的分发行为，EventBus中提供了3个默认的Dispatcher实现，分别用于不同场景的事件分发:`ImmediateDispatcher`，`LegacyAsyncDispatcher`，`PerThreadQueuedDispatcher`

EventBus 是同步的，AsyncEventBus 是异步的，可以根据自己的需要选择使用。

最后，EventBus中使用了非常多的数据结构，比如MultiMap、CopyOnWriteArraySet等，还有一些缓存和映射的工具库，这些大部分都来自于Guava。