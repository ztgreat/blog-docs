Java NIO 由以下几个核心部分组成：

1、Buffer

2、Channel

3、Selector

传统的IO操作面向数据流，面向流 的 I/O 系统一次一个字节地处理数据，意味着每次从流中读一个或多个字节，直至完成，数据没有被缓存在任何地方。

NIO操作面向缓冲区（ 面向块），数据从Channel读取到Buffer缓冲区，随后在Buffer中处理数据。

## 什么是Buffer（缓冲区）？
Buffer 是一个对象， 它包含一些要写入或者刚读出的数据。在面向流的 I/O 中，一般将数据直接写入或者将数据直接读到 Stream 对象中。

缓冲区实质上是一个数组。通常它是一个字节数组，内部维护几个状态变量，可以实现在同一块缓冲区上反复读写（不用清空数据再写）。

## Buffer（缓冲区） 数据结构

Buffer 实质上是一个数组，实现其功能的关键点在于几个状态变量:

1、mark：初始值为-1，用于备份当前的position;

2、position：初始值为0，position表示当前可以写入或读取数据的位置，当写入或读取一个数据后，position向前移动到下一个位置；

3、limit：**写模式下**，limit表示最多能往Buffer里写多少数据，等于capacity值；**读模式下**，limit表示最多可以读取多少数据，小于等于 capacity 值。

4、capacity：缓存数组大小

下面演示Buffer 的读写过程以及一些操作状态变量的方法：

**初始状态**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151016189.png)

**第一次写入数据**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151022563.png)

**第二次写入数据**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151029874.png)

**flip 方法**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151037383.png)

```
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

**第一次读取数据**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151045672.png)

**第二次读取数据**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151053187.png)

**clear 方法**

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807151058556.png)


```
 public final Buffer clear() {
     position = 0;
     limit = capacity;
     mark = -1;
     return this;
 }
```

### Buffer 重要方法
**mark()**：把当前的position赋值给mark

```
/**
 * Sets this buffer's mark at its position.
 *
 * @return  This buffer
 */
public final Buffer mark() {
    mark = position;
    return this;
}
```
**flip()**：Buffer有两种模式，写模式和读模式，flip后Buffer从写模式变成读模式（设置状态变量值）。

```
/**
 * Flips this buffer.  The limit is set to the current position and then
 * the position is set to zero.  If the mark is defined then it is
 * discarded.
 */
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

**reset()**：把mark值还原给position

```
/**
 * Resets this buffer's position to the previously-marked position.
 * @return  This buffer
 */
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
```
**clear()**：读完Buffer中的数据，需要让Buffer准备好再次被写入，clear会恢复状态值，但不会擦除数据。

```
 /**
  * Clears this buffer.  The position is set to zero, the limit is set to
  * the capacity, and the mark is discarded.
  * <p> This method does not actually erase the data in the buffer, but it
  * is named as if it did because it will most often be used in situations
  * in which that might as well be the case. </p>
  * @return  This buffer
  */
 public final Buffer clear() {
     position = 0;
     limit = capacity;
     mark = -1;
     return this;
 }
```

**rewind()**：重置position为0，从头读写数据。

```
/**
 * Rewinds this buffer.  The position is set to zero and the mark is
 * discarded.
 * @return  This buffer
 */
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```
## Buffer 类型
每一种基本 Java 类型都有一种缓冲区类型：

 - ByteBuffer
 - CharBuffer
 - ShortBuffer
 - IntBuffer
 - LongBuffer
 - FloatBuffer
 - DoubleBuffer
 - MappedByteBuffer

最常用的缓冲区类型是 ByteBuffer。]每一个 Buffer 类都是 Buffer 接口的一个实例。 除了 ByteBuffer，每一个 Buffer 类都有完全一样的操作，只是它们所处理的数据类型不一样。因为大多数标准 I/O 操作都使用 ByteBuffer，所以它具有所有共享的缓冲区操作以及一些特有的操作。

![这里写图片描述](http://img.blog.ztgreat.cn/document/nio/20180807111751984.png)

### ByteBuffer

ByteBuffer的实现类包括HeapByteBuffer与DirectByteBuffer,顾名思义，一个是堆字节缓存，一个是直接缓存


### HeapByteBuffer
在原理上，可以看出分配的buffer是在heap区域的，其实真正flush到远程的时候会先拷贝得到OS内核

### DirectByteBuffer
另一种有用的 ByteBuffer 是直接缓冲区。 直接缓冲区 是为加快 I/O 速度，而以一种特殊的方式分配其内存的缓冲区。


```
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
DirectByteBuffer(int cap) {
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        //申请内存
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    //初始化
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

DirectByteBuffer通过unsafe.allocateMemory申请堆外内存，并在ByteBuffer的address变量中维护指向该内存的地址。

实际上，直接缓冲区的准确定义是与实现相关的。oracle的文档是这样描述直接缓冲区的：

>  Given a direct byte buffer, the Java virtual machine will make a best effort to perform native I/O operations directly upon it. That is, it will attempt to avoid copying the buffer's content to (or from) an intermediate buffer before (or after) each invocation of one of the underlying operating system's native I/O operations.
>  The contents of direct buffers may reside outside of the normal garbage-collected heap


**给定一个直接字节缓冲区，Java 虚拟机将尽最大努力直接对它执行本机 I/O 操作。也就是说，它会在每一次调用底层操作系统的本机 I/O 操作之前(或之后)，尝试避免将缓冲区的内容拷贝到一个中间缓冲区中(或者从一个中间缓冲区中拷贝数据)，直接缓冲区的内容可能位于正常的垃圾收集堆之外。**

## 什么是通道（Channel）？
Channel是一个对象，可以通过它读取和写入数据。

通常我们都是将数据写入包含一个或者多个字节的缓冲区，然后再将缓存区的数据写入到通道中，将数据从通道读入缓冲区，再从缓冲区获取数据。

Channel 类似于原I/O中的流（Stream），但有所区别：

1、流是单向的，通道是双向的，可读可写。

2、流读写是阻塞的，通道可以异步读写。

目前Channel主要实现类有：

1、FileChannel

2、DatagramChannel

3、SocketChannel

4、ServerSocketChannel

### FileChannel
Java NIO中的FileChannel是一个连接到文件的通道。可以通过文件通道读写文件。

**read:** 从文件中指定的位置开始读取数据到缓冲区

```
  /**
   * Reads a sequence of bytes from this channel into a subsequence of the
   * given buffers.
   *
   * Bytes are read starting at this channel's current file position, and
   * then the file position is updated with the number of bytes actually
   * read.  Otherwise this method behaves exactly as specified in the interface.
   * 
   */
  public abstract long read(ByteBuffer[] dsts, int offset, int length)
      throws IOException;
```
**write：** 将缓冲区的数据写入文件，从文件指定处开始。
```
    /**
     * Writes a sequence of bytes to this channel from a subsequence of the
     * given buffers.
     *
     * Bytes are written starting at this channel's current file position
     * unless the channel is in append mode, in which case the position is
     * first advanced to the end of the file.  The file is grown, if necessary,
     * to accommodate the written bytes, and then the file position is updated
     * with the number of bytes actually written.  Otherwise this method
     * behaves exactly as specified in the interface. 
     * 
     */
    public abstract long write(ByteBuffer[] srcs, int offset, int length)
        throws IOException;
```

### DatagramChannel
Java NIO 中的 DatagramChannel 是一个能收发 UDP 包的通道。它发送和接收的是数据包。

### SocketChannel

Java NIO 中的 SocketChannel 是一个连接到 TCP 网络套接字的通道
### ServerSocketChannel
Java NIO 中的 ServerSocketChannel 是一个可以监听新进来的 TCP 连接的通道。

## 参考
[NIO 入门](https://www.ibm.com/developerworks/cn/education/java/j-nio/j-nio.html)

[深入浅出NIO之Channel、Buffer](https://www.jianshu.com/p/052035037297)

[docs.oracle.com](https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html)