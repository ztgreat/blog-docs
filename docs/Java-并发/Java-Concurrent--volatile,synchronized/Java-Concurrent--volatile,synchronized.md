在多线程并发编程中**synchronized**和**volatile**大家基本都很常见，**volatile**是轻量级的**synchronized**(**为什么？**)
**volatile**在多处理器开发中保证了共享变量的可见性，可见性的意思就是说**当一个线程修改一个共享变量时，另一个线程能读到这个修改的值**
**volatile**它不会引起线程上下文的切换和调度。
其实在说**volatile**之前，需要说一下Java内存模型，我个人理解：Java是跨平台的，它能跨平台原因在于java虚拟机的存在，通过java虚拟机屏蔽掉了底层硬件细节，这样面向开发者而言都是一样的，（**深入理解java虚拟机上是这样说的**）Java虚拟机规范中定义了一种Java内存模型（Java Memmory Model,JMM）来屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访问效果。
可以参考  [深入理解Java内存模型（一）——基础](http://www.infoq.com/cn/articles/java-memory-model-1)
这篇文章也是 **Java并发编程的艺术**中的内容
### (一) volatile

这里假设大家已经对Java内存模型有了一个大概的认识。
**volatile变量具有两个特性**：

 - **可见性** 当一个线程修改了这个变量的值，新值对于其他线程来说是可以立即得到的，而普通变量则不行，普通变量的值在线程间传递均需要通过主内存来完成。
 - **禁止指令重排序优化**，普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。
    有volatile变量修饰的共享变量进行**写操作**的时候会加上lock前缀，Lock前缀的指定在多核处理器下会引发下面两件事：

 **1）将当前处理器缓存行的数据写回到系统内存中**

 **2）这个写回内存的操作会使其他CPU里缓存了该地址的数据无效（保证了可见性）**

 上面的描述中摘自 Java并发编程的艺术艺术。

既然我们知道了Java内存模型保证了volatile的可见性，那是不是**volatile的运算就是线程安全的呢？** 来看看 深入理解Java虚拟机中给出的一个例子：

```
public class VolatileTest {
    
    public  static  volatile  int race=0;
    public  static  void increase(){
        race=race+1;
        //race++;
    }
    private  static  final  int THREADS_COUNT=20;
    public static void main(String[] args)throws  Exception{

        Thread[] threads=new Thread[THREADS_COUNT];
        for (int i=0;i<THREADS_COUNT;i++){
            threads[i]=new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j=0;j<1000;j++){
                        increase();
                    }
                }
            });
            threads[i].start();

        }
        while (Thread.activeCount()>2){
            Thread.yield();
        }
        System.out.println(race);
    }
}
```
运行输出的结果并不一定就是1000*20；
 其问题在于race=race+1;（race++），这个操作并不是一个原子操作，首先会先取出数据，然后再执行操作，最后在写会内存，我在CAS中说了，就算是原子操作代码如果组合成复合操作，那么其原子性就得不到保障，volitile只保证了可见性，也就是说这一刻线程读到的数据是正确的，当读出数据到栈后，也许数据就被其它线程修改了，该线程看得到了volitile的改变(高速缓存中的值失效)，但是该线程此时操作的是临时变量（栈中的值），然后再将过期的值写会了内存，导致了结果不正确,下面来看看increase()方法的字节码：

 ![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726123503391.png)

 具体解释就：
 **(1)getfield 把race读到栈顶**

 **(2)iconst_1把1放到栈顶**

 **(3)iadd 将栈顶两个元素相加放到栈顶**

 **(4)putfield 将栈顶元素写回race**



### (二) synchronized

synchronized可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性。
synchronized 怎么实现的呢？ 其实就是通过锁来控制的，只是这个锁是编译器帮我们加上的。

 - **Java中的每一个对象都可以作为锁。**
 - **对于同步方法，锁是当前实例对象。**
 - **对于静态同步方法，锁是当前对象的Class对象。**
 - **对于同步方法块，锁是Synchonized括号里配置的对象**。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。

JVM规范规定JVM基于进入和退出**Monitor对象**来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现，而**方法同步是使用另外一种方式实现的**，细节在JVM规范里并没有详细说明，但是方法的同步同样可以使用这两个指令来实现。monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处， JVM要保证每个monitorenter必须有对应的monitorexit与之配对。**任何对象都有一个 monitor 与之关联，当且一个monitor 被持有后，它将处于锁定状态**。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁，来看一个实例：
####  synchronized 修饰方法块（用类作为锁）
```
public class SynchronizedTest  {

    public synchronized void add(){
    }
    public  void insert(){
        synchronized (SynchronizedTest.class){
        }
    }
}
```
字节码输出结果：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726135054815.png)

从上面可以看出，**同步代码块**是使用monitorenter和monitorexit指令实现的
这里将**SynchronizedTest.class 作为了锁对象，这个在整个内存中只会存在一份，所以这个锁是正确的，当然也可以用this，但是this指代的是一个具体对象，这样对同一个类的同一个实例对象来说是线程安全的，对同一个类的不同实例对象是互相独立的**，来看看例子：

####   一、synchronized 修饰方法块（用类实例作为锁）
```
public class SynchronizedTest  {

    public  void insert(){
        //这里是this
        synchronized (this){
            try {
                while (true)
                    Thread.sleep(100000);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args)throws  Exception{
        Thread.currentThread().setName("main thread");
        final  SynchronizedTest synchronizedTest1=new SynchronizedTest();
        final  SynchronizedTest synchronizedTest2=new SynchronizedTest();
        new Thread(new Runnable() {
            public void run() {
                synchronizedTest1.insert();
            }
        },"child thread "+1).start();

        new Thread(new Runnable() {
            public void run() {
                synchronizedTest2.insert();
            }
        },"child thread "+2).start();
        while (true)
            Thread.sleep(100000);
    }
}
```
我们通过jstack工具来分析线程状态：
**main线程 sleep状态 **：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726144220265.png)



**child thread 1 状态sleep 状态**

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726144349016.png)



**child thread 2 状态sleep 状态**

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726144414864.png)



接下来把this 改成 **SynchronizedTest.class**

```
synchronized (SynchronizedTest.class)
```
**child thread 1 状态sleep 状态**

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726144808144.png)



**child thread 2 状态block状态**

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170726144820616.png)



验证了我们所说的是正确的。

####  二、synchronized 修饰非静态方法

```
public class Father {
    public  synchronized void method(){
        System.out.println("father");
    }
}
```
使用 synchronized 修饰非静态方法，实际是用的实例对象作为锁，也就是不同实例可以并发执行该同步方法，同一个实例串行执行该同步方法，可以用上面类似的方法进行实践证明


#### 三、synchronized 修饰静态方法
静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象（也就是用了类作为锁）。

```
public class SynchronizedTest  {

    public synchronized static void insert(){
        System.out.println("insert");
        while (true){
        }
    }
    public static void main(String[] args)throws  Exception{
        final  SynchronizedTest synchronizedTest1=new SynchronizedTest();
        final  SynchronizedTest synchronizedTest2=new SynchronizedTest();
        new Thread(new Runnable() {
            public void run() {
                synchronizedTest1.insert();
            }
        },"child thread "+1).start();

        new Thread(new Runnable() {
            public void run() {
                synchronizedTest2.insert();
            }
        },"child thread "+2).start();
        while (true)
            Thread.sleep(100000);
    }
}
```
上面代码运行 只会输出一个 insert,表明了在串行执行insert 方法，如果去掉static关键字，则会并行执行insert方法。



####  四、继承中的synchronized 

```
public class Father {
    public  synchronized void method(){
        System.out.println("father");
    }
}
```

```
public class Child extends  Father {
    @Override
    public synchronized void method() {

        System.out.println("child");
        super.method();
    }

    public static void main(String[] args){
        final  Child child=new Child();
        Thread a=   new Thread(new Runnable() {
                public void run() {
                    child.method();
                }
            });
        Thread b=   new Thread(new Runnable() {
            public void run() {
                child.method();
            }
        });
        a.start();
        b.start();
    }
}
```
1.首先我们将Child 中的method 改成如下样子：

```
    public  void method() {
        System.out.println("child");
        while (true){

        }
      //  super.method();
    }
```
运行结果：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170919134503093.png)

**结论：同一个实例，子类重写父类同步方法，子类不加synchronized 关键字，在子类method 方法中并行执行**。

2.再加上 synchronized 关键字

```
    public synchronized void method() {
        System.out.println("child");
        while (true){

        }
      //  super.method();
    }
```
![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170919134730925.png)

**结论：同一个实例，子类重写父类同步方法，子类加synchronized 关键字，在子类method 方法中串行执行**，当然也可以用前面的方法，用jstack来分析线程的状态。

3. 再次改造 method 方法
  这是child 里面的method 方法：
```
    public  void method() {
        System.out.println("child");
        super.method();
    }
```
这是father里面的method 方法：

```
    public  synchronized void method(){
        System.out.println("father");
        while (true){

        }
    }
```
运行结果：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170919135236482.png)



**结论：同一个实例，子类重写父类同步方法，子类不加synchronized 关键字，在child 的method 方法中并行，在father 的method 方法中串行**。

4. 在3 的基础上，我们改变main 方法

```
    public static void main(String[] args){

        final  Child child=new Child();
        final  Child child2=new Child();
         Thread a=   new Thread(new Runnable() {
                public void run() {
                    child.method();
                }
            });
        Thread b=   new Thread(new Runnable() {
            public void run() {
                child2.method();
            }
        });
        a.start();
        b.start();
    }
```
运行结果：

![这里写图片描述](http://img.blog.ztgreat.cn/document/juc/20170919140138262.png)

可以看到，**在不同实例下，子类重写父类同步方法，子类不加synchronized 关键字，父类同步方法并行执行**。

通过上面实验我们知道了：**synchronized 关键字是不会继承的**，子类必须要显示声明synchronized 才会具备同步特性  如果子类不显示声明的话  ，它是不会有同步锁的，如果里面调用了父类的同步方法，那么在调用的时候会获取锁，锁对象是当前实例。

**synchronized**是用的锁来实现同步的，那么锁又放在哪里的呢？----锁是存在Java对象头里的

如果对象是数组类型，则虚拟机用3个字宽（word）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。

|长度|内容|说明
-----|---|---|
32/64bit|Mark Word| 存储对象的hashCode或锁信息等
32/64bit|Class Metadata　Address| 存储到对象类型数据的指针
32/64bit| Array length | 数组的长度（如果当前对象是数组）
当要进入synchronized修饰的语句块时会检查当前对象头中的Mark Word信息，如果没有锁标记，则进行相应标记然后进入，如果有锁标记，查看锁的所有者是否是当前线程，是则直接进入（**synchronized可重入**），否则阻塞。大致流程就是这样的，具体细节暂时不用研究，一下钻太深了容易混，先有面的认识，再具体到点上，后面会介绍锁的种类与升级。


####  **synchronized 总结**
1. 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；
2. 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象；
3. 修饰一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象；
4. 修饰一个类（用类作为锁对象），其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象。
5. synchronized 关键字 不会继承

**参考资料：**

**Java 并发编程的艺术**

**深入理解Java虚拟机**

**[死磕Java并发-----深入分析synchronized的实现原理](http://blog.csdn.net/chenssy/article/details/54883355)**

**[如何使用jstack分析线程状态](http://www.jianshu.com/p/6690f7e92f27)**

**[深入理解Java内存模型（一）——基础](http://ifeve.com/java-memory-model-1/)**

**[聊聊并发（二）Java SE1.6中的Synchronized](http://ifeve.com/java-synchronized/)**