背景：

现在的互联网环境下，分布式系统大行其道，而分布式系统的根基在于网络编程，而 Netty 恰恰是 Java 领域网络编程的王者。如果要致力于开发高性能的服务器程序、高性能的客户端程序，必须掌握 Netty，而本课程的目的就是带领你进入基于 Netty 的网络编程世界。本课程从基础 NIO 编程开始，到 Netty 入门及进阶，参数优化到源码分析，由浅入深，为 Netty 学习打下坚实基础。本课程的目标是带你 Netty 入门，理解其基本运行原理和高效原因，并具备一定的 Netty 编码能力。



# 1，NIO基础

non-blocking io 非阻塞 IO

## 1，三大组件

### 1.1， Channel & Buffer

- channel：读写数据的**双向通道**
- buffer：缓存
- channel可以把数据读取到buffer，也可以将buffer的数据写入到channel

```mermaid
graph LR
channel --> buffer
buffer --> channel
```

常见的 Channel 有

* FileChannel
* DatagramChannel
* SocketChannel
* ServerSocketChannel

buffer 则用来缓冲读写数据，常见的 buffer 有

* ByteBuffer
  * MappedByteBuffer
  * DirectByteBuffer
  * HeapByteBuffer
* ShortBuffer
* IntBuffer
* LongBuffer
* FloatBuffer
* DoubleBuffer
* CharBuffer

### 1.2 Selector（选择器）



- ## 服务器的演化

  - ### 多线程版设计

    - ```mermaid
      graph TD
      subgraph 多线程版
      t1(thread) --> s1(socket1)
      t2(thread) --> s2(socket2)
      t3(thread) --> s3(socket3)
      end
      ```

    - 多线程处理，每个线程处理一个连接。（类似饭店吃饭，一名服务员会服务一个客人，多个客人就多名服务员）

    - ###### ==缺点==

      - 多线程占用内存高，（每个链接都占用一个线程）
    - 线程上下文切换成本高（因为太多的线程需要切换，所以切换频繁，消耗性能）
      - **只适合连接数少的场景**

  - ### 线程池版设计

    - ```mermaid
      graph TD
      subgraph 线程池版
      t4(thread) --> s4(socket1)
      t5(thread) --> s5(socket2)
      t4(thread) -.-> s6(socket3)
      t5(thread) -.-> s7(socket4)
      end
      ```
  
      （类似，一名服务员服务客人，必须服务完该客人点餐吃饭走后，才能去服务其他人）
  
    - ==缺点==
  
      - 线程池为**阻塞模式**，即（一个线程完全处理完该链接才能处理下一个链接）
      - 仅适合短链接的情况（会很快断开连接，以提高线程的使用率）
  
  - ### Selector 版设计
  
    - selector的作用就是配合一个线程管理多个channel，获取这些channel上发送的事件，channel是处于非阻塞模式下，有事件时，selector会发现，交给线程处理。**适合连接数特别多，但流量低的场景（low traffic）**（如果流量高的情况，则多个channel都请求处理，则一个线程只能执行一个请求，所以不适用）
  
    - ```mermaid
      graph TD
      subgraph selector 版
      thread --> selector
      selector --> c1(channel)
      selector --> c2(channel)
      selector --> c3(channel)
      end
      ```
  
    - 调用 selector 的 select() 会阻塞直到 channel 发生了读写就绪事件，这些事件发生，select 方法就会返回这些事件交给 thread 来处理



## 2，ByteBuffer

###  2.1 ByteBuffer的正确使用

```java
@Slf4j
public class ChannelDemo1 {
    public static void main(String[] args) {
        try (RandomAccessFile file = new RandomAccessFile("helloword/data.txt", "rw")) {
            FileChannel channel = file.getChannel();
            ByteBuffer buffer = ByteBuffer.allocate(10);
            do {
                // 向 buffer 写入
                int len = channel.read(buffer);
                log.debug("读到字节数：{}", len);
                if (len == -1) {
                    break;
                }
                // 切换 buffer 读模式
                buffer.flip();
                while(buffer.hasRemaining()) {
                    log.debug("{}", (char)buffer.get());
                }
                // 切换 buffer 写模式
                buffer.clear();
            } while (true);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```



1. 读取文件的数据，向buffer中写入数据，例如调用 channel.read(buffer)
2. 调用flip（）切换至读模式
3. 从buffer读取数据，例如调用 buffer.get()
4. 调用clear（）或者 compact() 切换至**写模式**
5. 重复 1~4 步骤



### 2.2 ByteBuffer 结构

ByteBuffer有三个重要属性变量

- capacity
- position
- limit

开始时，如图。

![image-20230228085014774](Netty.assets/image-20230228085014774-16775454161721.png)

**开始为写模式**，position为写入位置，limit等于capacity容量，下图表示写入了 4 个字节后的状态

![image-20230228085159735](Netty.assets/image-20230228085159735-16775455224983.png)

调用flip（）方法后，**ByteBuffer进行读模式**，position切换到起始位置，limit切换为读取限制

![image-20230228085333539](Netty.assets/image-20230228085333539-16775456165775.png)

读取 4 个字节后，状态

![](Netty.assets/0020-16775456314898.png)

调用clear()方法，重新进入读模式

![image-20230228085415777](Netty.assets/image-20230228085415777-167754565707910.png)

调用compact()方法，则是将未读完的数据向前压缩，然后切换至写模式。（clear()方法是直接全部清除，重新写入）

![image-20230228085504992](Netty.assets/image-20230228085504992-167754570629912.png)

**注意：compact（）方法是将未读完的数据全部向前迁移，例：[A,B,C,D],读模式下，读取[A,B]，调用compact（）方法迁移后，实际为[C,D,C,D]。但是写的位置是从数组[2]，即第二个C的位置开始操作，也就是会覆盖向前迁移后遗留下的byte**



### 2.3 ByteBuffer 常见方法

#### 分配空间

- Bytebuffer buf = ByteBuffer.allocate(16);   class java.nio.HeapByteBuffer
- ByteBuffer.allocateDirect(16)；                     class java.nio.DirectByteBuffer  

两种方法属于不同的类，可以看出分别为堆内存和直接内存

- HeapByteBuffer：使用java堆内存，读写效率低，会受到GC影响（GC情况下会产生停顿以及垃圾回收后，为减少内存碎片会进行内存数据的迁移，都会造成效率的底下）
- DirectByteBuffer：使用系统的直接内存，读写效率高（因为首先是读数据到系统内存，然后再从系统内存到java管理的内存，而使用直接内存少了一次拷贝，效率必然高），不会受到GC的影响，分配效率低

#### 向 buffer 写入数据

- 调用 channel 的 read 方法
- 调用 buffer 自己的 put 方法

```java
int readBytes = channel.read(buf);
```

和

```java
buf.put((byte)127);
```

#### 从 buffer 读取数据

同样有两种办法

* 调用 channel 的 write 方法
* 调用 buffer 自己的 get 方法

```java
int writeBytes = channel.write(buf);
```

和

```java
byte b = buf.get();
```

get（）方法会使得position读指针后移，如果需要从头读取数据

- 可以调用 rewind 方法将 position 重新置为 0
- 或者调用 get(int i) 方法获取索引 i 的内容，**它不会移动读指针**



#### mark 和 reset

mark 是在读取时，做一个标记，即使 position 改变，只要调用 reset 就能回到 mark 的位置

> **注意**
>
> rewind 和 flip 都会清除 mark 位置

#### 字符串与 ByteBuffer 互转

Charset会直接切换到读模式，而put方法不会

```java
buffer.put("hello".getBytes());

ByteBuffer buffer1 = StandardCharsets.UTF_8.encode("你好");
ByteBuffer buffer2 = Charset.forName("utf-8").encode("你好");

debug(buffer1);
debug(buffer2);

CharBuffer buffer3 = StandardCharsets.UTF_8.decode(buffer1);
System.out.println(buffer3.getClass());
System.out.println(buffer3.toString());
```

#### ⚠️ Buffer 的线程安全

> Buffer 是**非线程安全的**

#### 2.4 Scattering Reads 分散读取

```java
try (RandomAccessFile file = new RandomAccessFile("helloword/3parts.txt", "rw")) {
    FileChannel channel = file.getChannel();
    ByteBuffer a = ByteBuffer.allocate(3);
    ByteBuffer b = ByteBuffer.allocate(3);
    ByteBuffer c = ByteBuffer.allocate(5);
    channel.read(new ByteBuffer[]{a, b, c});
    a.flip();
    b.flip();
    c.flip();
    debug(a);
    debug(b);
    debug(c);
} catch (IOException e) {
    e.printStackTrace();
}
```

## 3，文件编程

















































