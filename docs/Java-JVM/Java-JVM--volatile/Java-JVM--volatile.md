

>转载 占小狼 [java volatile关键字解惑](https://www.jianshu.com/p/506c1e38a922)

# 前言

网上看到这篇关于volatile的文章，个人觉得写得很好，前前后后还是看过很多次了，有深度的，自己也没有钻研这么深，转载过来，吸收吸收，变成自己的.

> 注：文章略有改动

# volatile特性

内存可见性：通俗来说就是，线程A对一个volatile变量的修改，对于其它线程来说是可见的，即线程每次获取volatile变量的值都是最新的。

# volatile的使用场景

通过关键字sychronize可以防止多个线程进入同一段代码，在某些特定场景中，volatile相当于一个轻量级的sychronize，因为不会引起线程的上下文切换，但是使用volatile必须满足两个条件：

1、对变量的写操作不依赖当前值，如多线程下执行a++，是无法通过volatile保证结果准确性的(a++ 是一个复合操作)；

2、该变量没有包含在具有其它变量的不变式中，这句话有点拗口，看代码比较直观。

```
public class NumberRange {
    private volatile int lower = 0;
     private volatile int upper = 10;

    public int getLower() { return lower; }
    public int getUpper() { return upper; }

    public void setLower(int value) { 
        if (value > upper) 
            throw new IllegalArgumentException(...);
        lower = value;
    }

    public void setUpper(int value) { 
        if (value < lower) 
            throw new IllegalArgumentException(...);
        upper = value;
    }
}

```

上述代码中，上下界初始化分别为0和10，假设线程A和B在某一时刻同时执行了setLower(8)和setUpper(5)，且都通过了不变式的检查，设置了一个无效范围（8, 5），所以在这种场景下，需要通过sychronize保证方法setLower和setUpper在每一时刻只有一个线程能够执行。

下面是我们在项目中经常会用到volatile关键字的两个场景：

1、**状态标记量**
在高并发的场景中，通过一个boolean类型的变量isopen，控制代码是否走促销逻辑，该如何实现？

```
public class ServerHandler {
    private volatile isopen;
    public void run() {
        if (isopen) {
           //促销逻辑
        } else {
          //正常逻辑
        }
    }
    public void setIsopen(boolean isopen) {
        this.isopen = isopen
    }
}

```

场景细节无需过分纠结，这里只是举个例子说明volatile的使用方法，用户的请求线程执行run方法，如果需要开启促销活动，可以通过后台设置，具体实现可以发送一个请求，调用setIsopen方法并设置isopen为true，由于isopen是volatile修饰的，所以一经修改，其他线程都可以拿到isopen的最新值，用户请求就可以执行促销逻辑了.

2、**double check**
单例模式的一种实现方式，但很多人会忽略volatile关键字，因为没有该关键字，程序也可以很好的运行，只不过代码的稳定性总不是100%，说不定在未来的某个时刻，隐藏的bug就出来了。

```
class Singleton {
    private volatile static Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {
            syschronized(Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    } 
}

```

不过在众多单例模式的实现中，我比较推荐懒加载的优雅写法Initialization on Demand Holder（IODH）。

```
public class Singleton {  
    static class SingletonHolder {  
        static Singleton instance = new Singleton();  
    }  
      
    public static Singleton getInstance(){  
        return SingletonHolder.instance;  
    }  
}
```

当然，如果不需要懒加载的话，直接初始化的效果更好。

# 如何保证内存可见性？

在java虚拟机的内存模型中，有主内存和工作内存的概念，每个线程对应一个工作内存，并共享主内存的数据，下面看看操作普通变量和volatile变量有什么不同：

1、对于普通变量：读操作会优先读取工作内存的数据，如果工作内存中不存在，则从主内存中拷贝一份数据到工作内存中；写操作只会修改工作内存的副本数据，这种情况下，其它线程就无法读取变量的最新值。

2、对于volatile变量，读操作时JMM会把工作内存中对应的值设为无效，要求线程从主内存中读取数据；写操作时JMM会把工作内存中对应的数据刷新到主内存中，这种情况下，其它线程就可以读取变量的最新值。

volatile变量的内存可见性是基于内存屏障(Memory Barrier)实现的，什么是内存屏障？内存屏障，又称内存栅栏，是一个CPU指令。在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JMM为了保证在不同的编译器和CPU上有相同的结果，通过插入特定类型的内存屏障来禁止特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和CPU：不管什么指令都不能和这条Memory Barrier指令重排序。

这段文字显得有点苍白无力，不如来段简明的代码：

```
class Singleton {
    private volatile static Singleton instance;
    private int a;
    private int b;
    private int b;
    public static Singleton getInstance() {
        if (instance == null) {
            syschronized(Singleton.class) {
                if (instance == null) {
                    a = 1;  // 1
                     b = 2;  // 2
                    instance = new Singleton();  // 3
                    c = a + b;  // 4
                }
            }
        }
        return instance;
    } 
}

```

1、如果变量instance没有volatile修饰，语句1、2、3可以随意的进行重排序执行，即指令执行过程可能是3214或1324。

2、如果是volatile修饰的变量instance，会在语句3的前后各插入一个内存屏障。

通过观察volatile变量和普通变量所生成的汇编代码可以发现，操作volatile变量会多出一个lock前缀指令：

```
Java代码：
instance = new Singleton();

汇编代码：
0x01a3de1d: movb $0x0,0x1104800(%esi);
0x01a3de24: **lock** addl $0x0,(%esp);
```

这个lock前缀指令相当于上述的内存屏障，提供了以下保证：

1、将当前CPU缓存行的数据写回到主内存；

2、这个写回内存的操作会导致在其它CPU里缓存了该内存地址的数据无效。

CPU为了提高处理性能，并不直接和内存进行通信，而是将内存的数据读取到内部缓存（L1，L2）再进行操作，但操作完并不能确定何时写回到内存，如果对volatile变量进行写操作，当CPU执行到Lock前缀指令时，会将这个变量所在缓存行的数据写回到内存，不过还是存在一个问题，就算内存的数据是最新的，其它CPU缓存的还是旧值，所以为了保证各个CPU的缓存一致性，每个CPU通过嗅探在总线上传播的数据来检查自己缓存的数据有效性，当发现自己缓存行对应的内存地址的数据被修改，就会将该缓存行设置成无效状态，当CPU读取该变量时，发现所在的缓存行被设置为无效，就会重新从内存中读取数据到缓存中。



# 深入分析 volatile

一下内容会涉及到一些汇编方面的内容，如果多看几遍，应该能看懂。

## 重排序

为了理解重排序，先看一段简单的代码

```
public class VolatileTest {

    int a = 0;
    int b = 0;

    public void set() {
        a = 1;
        b = 1;
    }

    public void loop() {
        while (b == 0) continue;
        if (a == 1) {
            System.out.println("i'm here");
        } else {
            System.out.println("what's wrong");
        }
    }
}

```

VolatileTest类有两个方法，分别是set()和loop()，假设线程B执行loop方法，线程A执行set方法，会得到什么结果？

答案是不确定，因为这里涉及到了编译器的重排序和CPU指令的重排序。

## 编译器重排序

编译器在不改变单线程语义的前提下，为了提高程序的运行速度，可以对字节码指令进行重新排序，所以代码中a、b的赋值顺序，被编译之后可能就变成了先设置b，再设置a。

因为对于线程A来说，先设置哪个，都不影响自身的结果。

## CPU指令重排序

CPU指令重排序又是怎么回事？
在深入理解之前，先看看x86的cpu缓存结构。

![20180914112553](http://img.blog.ztgreat.cn/document/juc/20180914112553.png)

1、各种寄存器，用来存储本地变量和函数参数，访问一次需要1cycle，耗时小于1ns；

2、L1 Cache，一级缓存，本地core的缓存，分成32K的数据缓存L1d和32k指令缓存L1i，访问L1需要3cycles，耗时大约1ns；

3、L2 Cache，二级缓存，本地core的缓存，被设计为L1缓存与共享的L3缓存之间的缓冲，大小为256K，访问L2需要12cycles，耗时大约3ns；

4、L3 Cache，三级缓存，在同插槽的所有core共享L3缓存，分为多个2M的段，访问L3需要38cycles，耗时大约12ns；

当然了，还有平时熟知的DRAM，访问内存一般需要65ns，所以CPU访问一次内存和缓存比较起来显得很慢。

对于不同插槽的CPU，L1和L2的数据并不共享，一般通过MESI协议保证Cache的一致性，但需要付出代价。

在MESI协议中，每个Cache line有4种状态，分别是：

1、M(Modified)
这行数据有效，但是被修改了，和内存中的数据不一致，数据只存在于本Cache中

2、E(Exclusive)
这行数据有效，和内存中的数据一致，数据只存在于本Cache中

3、S(Shared)
这行数据有效，和内存中的数据一致，数据分布在很多Cache中

4、I(Invalid)
这行数据无效

每个Core的Cache控制器不仅知道自己的读写操作，也监听其它Cache的读写操作，假如有4个Core：

1、Core1从内存中加载了变量X，值为10，这时Core1中缓存变量X的cache line的状态是E；

2、Core2也从内存中加载了变量X，这时Core1和Core2缓存变量X的cache line状态转化成S；

3、Core3也从内存中加载了变量X，然后把X设置成了20，这时Core3中缓存变量X的cache line状态转化成M，其它Core对应的cache line变成I（无效）

当然了，不同的处理器内部细节也是不一样的，比如Intel的core i7处理器使用从MESI中演化出的MESIF协议，F(Forward)从Share中演化而来，一个cache line如果是F状态，可以把数据直接传给其它内核，这里就不纠结了。

CPU在cache line状态的转化期间是阻塞的，经过长时间的优化，在寄存器和L1缓存之间添加了LoadBuffer、StoreBuffer来降低阻塞时间，LoadBuffer、StoreBuffer，合称排序缓冲(Memoryordering Buffers (MOB))，Load缓冲64长度，store缓冲36长度，Buffer与L1进行数据传输时，CPU无须等待。

1、CPU执行load读数据时，把读请求放到LoadBuffer，这样就不用等待其它CPU响应，先进行下面操作，稍后再处理这个读请求的结果。
2、CPU执行store写数据时，把数据写到StoreBuffer中，待到某个适合的时间点，把StoreBuffer的数据刷到主存中。

因为StoreBuffer的存在，CPU在写数据时，真实数据并不会立即表现到内存中，所以对于其它CPU是不可见的；同样的道理，LoadBuffer中的请求也无法拿到其它CPU设置的最新数据；

由于StoreBuffer和LoadBuffer是异步执行的，所以在外面看来，先写后读，还是先读后写，没有严格的固定顺序。

## 内存可见性如何实现

从上面的分析可以看出，其实是CPU执行load、store数据时的异步性，造成了不同CPU之间的内存不可见，那么如何做到CPU在load的时候可以拿到最新数据呢？

## 设置volatile变量

写一段简单的java代码，声明一个volatile变量，并赋值

```
public class VolatileTest {

    static volatile int i;

    public static void main(String[] args){
        i = 10;
    }
}
```

这段代码本身没什么意义，只是想看看加了volatile之后，编译出来的字节码有什么不同，执行 `javap -verbose VolatileTest` 之后，结果如下：

![20180914112829](http://img.blog.ztgreat.cn/document/juc/20180914112829.png)

让人很失望，没有找类似关键字synchronize编译之后的字节码指令（monitorenter、monitorexit），volatile编译之后的赋值指令putstatic没有什么不同，唯一不同是变量i的修饰flags多了一个`ACC_VOLATILE`标识。

不过，我觉得可以从这个标识入手，先全局搜下`ACC_VOLATILE`，无从下手的时候，先看看关键字在哪里被使用了，果然在accessFlags.hpp文件中找到类似的名字。

![20180914112921](http://img.blog.ztgreat.cn/document/juc/20180914112921.png)

通过`is_volatile()`可以判断一个变量是否被volatile修饰，然后再全局搜"is_volatile"被使用的地方，最后在`bytecodeInterpreter.cpp`文件中，找到putstatic字节码指令的解释器实现，里面有`is_volatile()`方法。

![20180914113006](http://img.blog.ztgreat.cn/document/juc/20180914113006.png)



当然了，在正常执行时，并不会走这段逻辑，都是直接执行字节码对应的机器码指令，这段代码可以在debug的时候使用，不过最终逻辑是一样的。

其中cache变量是java代码中变量i在常量池缓存中的一个实例，因为变量i被volatile修饰，所以`cache->is_volatile()`为真，给变量i的赋值操作由`release_int_field_put`方法实现。

再来看看`release_int_field_put`方法

![20180914113100](http://img.blog.ztgreat.cn/document/juc/20180914113100.png)

奇怪，在OrderAccess::release_store的实现中，第一个参数强制加了一个volatile，很明显，这是c/c++的关键字。

c/c++中的volatile关键字，用来修饰变量，通常用于语言级别的  [memory barrier](https://link.jianshu.com?t=http%3A%2F%2Fen.wikipedia.org%2Fwiki%2FMemory_barrier)，在"The C++ Programming Language"中，对volatile的描述如下：

>A volatile specifier is a hint to a compiler that an object may change its value in ways not specified by the language so that aggressive optimizations must be avoided.

volatile是一种类型修饰符，被volatile声明的变量表示随时可能发生变化，**每次使用时，都必须从变量i对应的内存地址读取，编译器对操作该变量的代码不再进行优化**，下面写两段简单的c/c++代码验证一下

```
#include <iostream>

int foo = 10;
int a = 1;
int main(int argc, const char * argv[]) {
    // insert code here...
    a = 2;
    a = foo + 10;
    int b = a + 20;
    return b;
}

```

代码中的变量i其实是无效的，执行`g++ -S -O2 main.cpp`得到编译之后的汇编代码如下：

![20180914113225](http://img.blog.ztgreat.cn/document/juc/20180914113225.png)

可以发现，在生成的汇编代码中，对变量a的一些无效负责操作果然都被**优化掉**了，如果在声明变量a时加上volatile

```
#include <iostream>

int foo = 10;
volatile int a = 1;
int main(int argc, const char * argv[]) {
    // insert code here...
    a = 2;
    a = foo + 10;
    int b = a + 20;
    return b;
}

```

再次生成汇编代码如下：

![20180914113337](http://img.blog.ztgreat.cn/document/juc/20180914113337.png)

和第一次比较，有以下不同：

1、对变量a赋值2的语句，也保留了下来，虽然是无效的动作，**所以volatile关键字可以禁止指令优化**，**其实这里发挥了编译器屏障的作用；**

编译器屏障可以避免编译器优化带来的内存乱序访问的问题，也可以手动在代码中插入编译器屏障，比如下面的代码和加volatile关键字之后的效果是一样

```
#include <iostream>

int foo = 10;
int a = 1;
int main(int argc, const char * argv[]) {
    // insert code here...
    a = 2;
    __asm__ volatile ("" : : : "memory"); //编译器屏障
    a = foo + 10;
    __asm__ volatile ("" : : : "memory");
    int b = a + 20;
    return b;
}

```

编译之后，和上面类似

![20180914113436](http://img.blog.ztgreat.cn/document/juc/20180914113436.png)



2、其中`_a(%rip)`是变量a的每次地址，通过`movl $2, _a(%rip)`可以把变量a所在的内存设置成2，关于RIP，可以查看 [x64下PIC的新寻址方式：RIP相对寻址](https://www.polarxiong.com/archives/x64%E4%B8%8BPIC%E7%9A%84%E6%96%B0%E5%AF%BB%E5%9D%80%E6%96%B9%E5%BC%8F-RIP%E7%9B%B8%E5%AF%B9%E5%AF%BB%E5%9D%80.html)

所以，每次对变量a的赋值，都会写入到内存中；每次对变量的读取，都会从内存中重新加载。

感觉有点跑偏了，让我们回到JVM的代码中来。

![20180914113632](http://img.blog.ztgreat.cn/document/juc/20180914113632.png)



执行完赋值操作后，紧接着执行`OrderAccess::storeload()`，这又是啥？

其实这就是经常会念叨的内存屏障，之前只知道念，却不知道是如何实现的。从CPU缓存结构分析中已经知道：一个load操作需要进入LoadBuffer，然后再去内存加载；一个store操作需要进入StoreBuffer，然后再写入缓存，这两个操作都是异步的，会导致不正确的指令重排序，所以在JVM中定义了一系列的内存屏障来指定指令的执行顺序。

JVM中定义的内存屏障如下，JDK1.7的实现

![20180914113738](http://img.blog.ztgreat.cn/document/juc/20180914113738.png)

1、loadload屏障（load1，loadload， load2）

2、loadstore屏障（load，loadstore， store）

这两个屏障都通过`acquire()`方法实现

![20180914113822](http://img.blog.ztgreat.cn/document/juc/20180914113822.png)



其中`__asm__`，表示汇编代码的开始。
volatile，之前分析过了，禁止编译器对代码进行优化。
把这段指令编译之后，发现没有看懂....最后的"memory"是编译器屏障的作用。

在LoadBuffer中插入该屏障，清空屏障之前的load操作，然后才能执行屏障之后的操作，可以保证load操作的数据在下个store指令之前准备好

3、storestore屏障（store1，storestore， store2）
通过"release()"方法实现：

![20180914113907](http://img.blog.ztgreat.cn/document/juc/20180914113907.png)



在StoreBuffer中插入该屏障，清空屏障之前的store操作，然后才能执行屏障之后的store操作，保证store1写入的数据在执行store2时对其它CPU可见。

4、storeload屏障（store，storeload， load）
对java中的volatile变量进行赋值之后，插入的就是这个屏障，通过"fence()"方法实现：

![20180914114024](http://img.blog.ztgreat.cn/document/juc/20180914114024.png)



看到这个有没有很兴奋？

通过`os::is_MP()`先判断是不是多核，如果只有一个CPU的话，就不存在这些问题了。

storeload屏障，完全由下面这些指令实现

```
__asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
```

为了试验这些指令到底有什么用，我们再写点c++代码编译一下

```
#include <iostream>

int foo = 10;

int main(int argc, const char * argv[]) {
    // insert code here...
    volatile int a = foo + 10;
    // __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
    volatile int b = foo + 20;

    return 0;
}
```

![20180914114137](http://img.blog.ztgreat.cn/document/juc/20180914114137.png)

从编译后的代码可以发现，第二次使用foo变量时，没有从内存重新加载，使用了寄存器的值。

把`__asm__ volatile ***`指令加上之后重新编译

![20180914114227](http://img.blog.ztgreat.cn/document/juc/20180914114227.png)

相比之前，这里多了两个指令，一个lock，一个addl。
**lock指令的作用是**：在执行lock后面指令时，会设置处理器的LOCK#信号（这个信号会锁定总线，阻止其它CPU通过总线访问内存，直到这些指令执行结束），**这条指令的执行变成原子操作，之前的读写请求都不能越过lock指令进行重排，相当于一个内存屏障**。

还有一个：第二次使用foo变量时，从内存中重新加载，保证可以拿到foo变量的最新值，这是由如下指令实现

```
__asm__ volatile ( : : : "cc", "memory");
```

同样是编译器屏障，通知编译器重新生成加载指令(不可以从缓存寄存器中取)。

**同时我们注意到对变量a,b的操作，并不是原子性的，只能保证是最新的，这个也是可见性的含义。**

## 读取volatile变量

同样在`bytecodeInterpreter.cpp`文件中，找到getstatic字节码指令的解释器实现。

![20180914114344](http://img.blog.ztgreat.cn/document/juc/20180914114344.png)

通过`obj->obj_field_acquire(field_offset)`获取变量值

![20180914114417](http://img.blog.ztgreat.cn/document/juc/20180914114417.png)



最终通过`OrderAccess::load_acquire`实现

```
inline jint OrderAccess::load_acquire(volatile jint* p) { return *p; }
```

底层基于C++的volatile实现，因为volatile自带了编译器屏障的功能，总能拿到内存中的最新值。

## 总结

volatile 的特性便是 内存可见性，被volatile修饰的变量会禁止重排序，重排序又分为编译器重排序和CPU指令重排序，其重排序的目的优化程序。

为了实现内存可见性，则是通过内存屏障来实现的，内存屏障说白了就是指令，强制命令cpu每次操作volatile变量时都从内存读取，修改后写回内存，这样其它线程便能及时的发现数据的更改，这便是可见性，但是volatile并不保证操作变量的原子性（在汇编代码中可以看出来），对于 对volatile的复合操作，如果需要保证原子性，那么仍然需要自己手动控制。
