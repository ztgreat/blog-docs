ClassLoader 顾名思义就是类加载器,**ClassLoader 作用**：

 - 负责将 Class 加载到 JVM 中
 - 审查每个类由谁加载（父优先的等级加载机制）
 - 将 Class 字节码重新解析成 JVM 统一要求的对象格式

# 类加载时机与过程
类从被加载到虚拟机内存中开始，直到卸载出内存为止，它的整个生命周期包括了：加载、验证、准备、解析、初始化、使用和卸载这7个阶段。其中，验证、准备和解析这三个部分统称为连接（linking）。

![这里写图片描述](http://img.blog.ztgreat.cn/document/jvm/20180805193923861.jpg)

其中，加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班的“开始”（仅仅指的是开始，而非执行或者结束，因为这些阶段通常都是互相交叉的混合进行，通常会在一个阶段执行的过程中调用或者激活另一个阶段），而解析阶段则不一定（它在某些情况下可以在初始化阶段之后再开始，这是为了支持Java语言的运行时绑定。


什么情况下需要开始类加载过程的第一个阶段:"加载"。虚拟机规范中并没强行约束，这点可以交给虚拟机的的具体实现自由把握，但是对于初始化阶段虚拟机规范是严格规定了如下几种情况，如果类未初始化会对类进行初始化：

1、创建类的实例

2、对类进行反射调用的时候，如果累没有进行过初始化，则需要先触发其初始化

3、当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化

4、当虚拟机启动时，用户需要指定一个要执行的主类（包含main 方法的那个类）,虚拟机会先初始化这个主类

5、当使用jdk1.7动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getstatic,REF_putstatic,REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行初始化，则需要先出触发其初始化。

**注意**，对于这五种会触发类进行初始化的场景，虚拟机规范中使用了一个很强烈的限定语：“有且只有”，这五种场景中的行为称为对一个类进行 主动引用。除此之外，所有引用类的方式，都不会触发初始化，称为 被动引用。

特别需要指出的是，**类的实例化与类的初始化**是两个完全不同的概念：

 - 类的实例化是指创建一个类的实例(对象)的过程；
 - 类的初始化是指为类中各个类成员(被static修饰的成员变量)赋初始值的过程，是类生命周期中的一个阶段。

 下面是被动引用的几个例子:
（1）
```
/**
 * jdk:1.8
 * 通过子类引用父类的静态字段,不会导致子类初始化
 */
class  SuperClass{

    static {
        System.out.println("SuperClass init!");
    }
    public  static  int value=123;

}
class  SubClass extends  SuperClass{

    static {
        System.out.println("SubClass init!");
    }

}

public class Test {
    public static void main(String[] args)throws  Exception{
      System.out.println(SubClass.value); 
      //输出:
      //SuperClass init!
      //123

    }
}

```
通过其子类来引用父类中定义的静态字段，只会触发父类初始化而不会触发子类的初始化。至于是否要触发子类的加载和验证，这个在虚拟机规范中并没有明确规定，这点取决于虚拟机的具体实现。

（2）

```
/**
 * jdk：1.8
 * 通过数组定义来引用类，不会触发此类的初始化
 */
class  SuperClass{

    static {
        System.out.println("SuperClass init!");
    }
    public  static  int value=123;

}
class  SubClass extends  SuperClass{

    static {
        System.out.println("SubClass init!");
    }

}

public class Test {
    public static void main(String[] args)throws  Exception{
        SuperClass[] sca=new SuperClass[10];
        //无输出
    }
}
```
通过数组定义来引用类，不会触发此类的初始化。

（3）

```
/**
 * jdk:1.8
 * 常量在编译阶段会存入调用类的常量池，
 * 本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化
 */
class ConstClass{
    static {
        System.out.println("ConstClass init!");
    }
    public static  final String HELLO = "hello";

}

public class Test {
    public static void main(String[] args)throws  Exception{
        System.out.println(ConstClass.HELLO);
        //输出 hello
    }
}
```
常量在编译阶段会存入调用类的常量池，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化

# ClassLoader 类结构分析

```
public abstract class ClassLoader {

  // The parent class loader for delegation
  // Note: VM hardcoded the offset of this field, thus all new fields
  // must be added *after* it.
  private final ClassLoader parent;
  
  //...
}
```
这里注意下有父加载器，这个我们再后面再来阐述，这里先有个印象。

以下是 ClassLoader 常用到的几个方法及其重载方法：

（1）
```
defineClass(String name, java.nio.ByteBuffer b,ProtectionDomain protectionDomain)
```
指定保护域（protectionDomain），把ByteBuffer的内容转换成 Java 类，这个方法被声明为final的。

（2）
```
defineClass(String name, byte[] b, int off, int len)
```

把字节数组 b中的内容转换成 Java 类，其开始偏移为off,这个方法被声明为final的。

（3）

```
//查找指定名称的类
findClass(String name) 
```

（4）

```
//加载指定名称的类
loadClass(String name) 
```

（5）
```
//链接指定的类
resolveClass(Class<?>) 
```

其中 defineClass 方法用来将 字节流解析成 JVM 能够识别的 Class 对象，有了这个方法意味着我们不仅仅可以通过 class 文件实例化对象，还可以通过其他方式实例化对象，如果我们通过网络接收到一个类的字节码，拿到这个字节码流直接创建类的 Class 对象形式实例化对象。如果直接调用这个方法生成类的 Class 对象，这个类的 Class 对象还没有 resolve ，这个 resolve 将会在这个对象真正实例化时才进行。

defineClass 通常是和findClass 方法一起使用的，我们通过覆盖ClassLoader父类的findClass 方法来实现类的加载规则，从而取得要加载类的字节码，然后调用defineClass方法生成类的Class 对象，如果你想在类被加载到JVM中时就被链接，那么可以接着调用另一个 resolveClass 方法，当然你也可以选择让JVM来解决什么时候才链接这个类。

# ClassLoader 的等级加载机制

Java默认提供的三个ClassLoader
## BootStrap ClassLoader
称为启动类加载器，是Java类加载层次中最顶层的类加载器，负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等，这个ClassLoader完全是JVM自己控制的，需要加载哪个类，怎么加载都是由JVM自己控制，别人也访问不到这个类，所以BootStrap ClassLoader不遵循委托机制(后面再阐述什么是委托机制)，没有子加载器。

下面是测试 BootStrap ClassLoader 加载的哪些文件：
```
/**
 * BootStrap ClassLoader 加载的文件
 */
public class Test {
    public static void main(String[] args)throws  Exception{
        System.out.println(System.getProperty("sun.boot.class.path"));
    }
}

```
输出：
```
C:\Program Files\Java\jre1.8.0_91\lib\resources.jar;
C:\Program Files\Java\jre1.8.0_91\lib\rt.jar;
C:\Program Files\Java\jre1.8.0_91\lib\sunrsasign.jar;
C:\Program Files\Java\jre1.8.0_91\lib\jsse.jar;
C:\Program Files\Java\jre1.8.0_91\lib\jce.jar;
C:\Program Files\Java\jre1.8.0_91\lib\charsets.jar;
C:\Program Files\Java\jre1.8.0_91\lib\jfr.jar;
C:\Program Files\Java\jre1.8.0_91\classes
```

## EtxClassLoader
称为扩展类加载器，负责加载Java的扩展类库，Java 虚拟机的实现会提供一个扩展库目录，该类加载器在此目录里面查找并加载 Java 类。默认加载JAVA_HOME/jre/lib/ext/目下的所有jar。

```
/**
 * EtxClassLoader 加载文件
 */
public class Test {
    public static void main(String[] args)throws  Exception{
        System.out.println(System.getProperty("java.ext.dirs"));
    }
}
```
输出：

```
C:\Program Files\Java\jre1.8.0_91\lib\ext;
```

## AppClassLoader

称为系统类加载器，负责加载应用程序classpath目录下的所有jar和class文件。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它。我们可以通过System.getProperty("java.class.path") 来查看 classpath。


除了引导类加载器（BootStrap ClassLoader）之外，所有的类加载器都有一个父类加载器，对于系统提供的类加载器来说，系统类加载器(如：AppClassLoader)的父类加载器是扩展类加载器（EtxClassLoader），而扩展类加载器的父类加载器是引导类加载器；对于开发人员编写的类加载器来说，其父类加载器是加载此类加载器 Java 类的类加载器。因为类加载器 Java 类如同其它的 Java 类一样，也是要由类加载器来加载的。一般来说，开发人员编写的类加载器的父类加载器是系统类加载器。类加载器通过这种方式组织起来，形成树状结构。树的根节点就是引导类加载器。


## ClassLoader加载类的原理
ClassLoader使用的是双亲委托模型来搜索加载类的
### 双亲委托模型
ClassLoader使用的是双亲委托机制来搜索加载类的，每个ClassLoader实例都有一个父类加载器的引用（**不是继承的关系，是一个组合的关系**），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

类加载器双亲委托模型：

![这里写图片描述](http://img.blog.ztgreat.cn/document/jvm/20180805222513969.png)

### 双亲委托模型好处
因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要 ClassLoader再加载一次。考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时就被引导类加载器（Bootstrcp ClassLoader）加载，所以用户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索类的默认算法。

### 类与类加载器
类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远远不限于类加载阶段。对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达更通俗一些：比较两个类是否"相等"，**只有再这两个类是有同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class 文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等**。

## 类加载器源码分析

### Launcher

```
public Launcher() {
    ExtClassLoader localExtClassLoader;
    try {
        // 扩展类加载器
        localExtClassLoader = ExtClassLoader.getExtClassLoader();
    } catch (IOException localIOException1) {
        throw new InternalError("Could not create extension class loader", localIOException1);
    }
    try {
        // 应用类加载器
        this.loader = AppClassLoader.getAppClassLoader(localExtClassLoader);
    } catch (IOException localIOException2) {
        throw new InternalError("Could not create application class loader", localIOException2);
    }
    // 设置AppClassLoader为线程上下文类加载器
    Thread.currentThread().setContextClassLoader(this.loader);
    // ...
    
    static class ExtClassLoader extends java.net.URLClassLoader
    static class AppClassLoader extends java.net.URLClassLoader
}
```
Launcher初始化了ExtClassLoader和AppClassLoader，并**将AppClassLoader设置为线程上下文类加载器**。

ExtClassLoader和AppClassLoader都继承自URLClassLoader，而最终的父类则为ClassLoader,看看它们的类层次：

 ![这里写图片描述](http://img.blog.ztgreat.cn/document/jvm/20180805223843888.png)

 ![这里写图片描述](http://img.blog.ztgreat.cn/document/jvm/20180805223853856.png)


初始化AppClassLoader时传入了ExtClassLoader实例，当我们进入源码跟踪，会来到URLClassLoader 中
```
public URLClassLoader(URL[] urls, ClassLoader parent,
                      URLStreamHandlerFactory factory) {
    super(parent);
    // this is to make the stack depth consistent with 1.1
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkCreateClassLoader();
    }
    ucp = new URLClassPath(urls, factory);
    acc = AccessController.getContext();
}
```
可以得知，**这个ExtClassLoader 作为了AppClassLoader 的parent**，在前面ClassLoader 源码中，我们知道有个parent 字段，这里就是在初始化这个字段。


### 双亲委派

要理解双亲委派，可以查看ClassLoader.loadClass方法:

```
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 检查是否已经加载过
        Class<?> c = findLoadedClass(name);
        if (c == null) { // 没有被加载过
            long t0 = System.nanoTime();
            // 先委派给父类加载器加载
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    //如果父加载器不存在，则委托给启动类加载器 加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // 如果父类加载器无法加载，自身才尝试加载
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```
先检查是否已经被加载过，若没有加载则调用父加载器的loadClass() 方法，**若父加载器为空，则默认使用启动类加载器作为父加载器**。如果父加载器失败，再调用自己的findClass 方法进行加载,因此到这里再次证明了类加载器的过程：

![这里写图片描述](http://img.blog.ztgreat.cn/document/jvm/20180805231028617.png)

### 打破 双亲委派

对于 ClassLoader的loadClass方法

1. findLoadedClass
2. 委托parent加载器加载（这里注意bootstrap加载器的parent为null)
3. 自行加载

如果我们需要打破这种双亲委派机制，那么继承ClassLoader覆盖loadClass方法，同时 打乱2和3的顺序，通过类名筛选自己要加载的类，其他的委托给parent加载器即可，不过如果你定义 java 内库中的类，最后也不会编译通过，因为java会在define方法中去检查该类路径，会报存在安全问题错误。

# 总结

关于类加载器网上有很多的文章，记录下来，也算是一个自己总结的过程，有时候看很多遍，不如实实在在的写一遍。

# 参考
深入理解Java 虚拟机

深入分析Java Web技术内幕

[深入理解JVM之ClassLoader](https://segmentfault.com/a/1190000013469223)

[详细深入分析 Java ClassLoader 工作机制](https://segmentfault.com/a/1190000008491597)

