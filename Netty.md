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





















































































