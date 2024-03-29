---
layout: post
title:  "Netty中的ByteBuf"
date:   2020-06-06 12:34:25 +0800
categories: netty
tags: netty ByteBuf ByteBuffer
subtitle: "网络编程相关知识"
toc: true
---



## Netty中的ByteBuf和Java NIO中ByteBuffer

### Java NIO中ByteBuffer中存在的问题

1. 长度固定, 一旦完成分配, 容量不能自动扩张或收缩, 当需要编码对象大于容量时, 会发生数组越界
2. 只有一个Position指针, 需要调用flip()和rewind等切换读写, 容易导致异常
3. API功能有限, 一些功能需要自己实现



### ByteBuf工作原理

- 参考JDK ByteBuffer的实现, 增加额外的功能, 解决缺点
- 聚合JDK ByteBuffer, 通过Facade模式对其进行包装, 可以减少自身代码量, 降低聚合成本
- 由于Netty底层调用了jdk nio, 所以ByteBuf 与 ByteBuffer 应该支持互相转换

### ByteBuf的实现

1. 使用两个指针 `readerIndex` , `writerIndex`避免切换

2. 封装扩容方法防止buffer溢出

   有两个阶段   

    < 4M 时,  以64为基数扩容2倍(64 -> 128 -> 256), 直到满足要求

   ```
   int newCapacity = 64;
   while (newCapacity < minNewCapacity) {
       newCapacity <<= 1;
   }
   ```

   \> 4M 是, 每次增加4M.



## DirectByteBuffer

Q：为什么需要用户进程(位于用户态中)要通过系统调用(Java中即使JNI)来调用内核态中的资源，或者说调用操作系统的服务了？
 A：intel cpu提供Ring0-Ring3四种级别的运行模式，Ring0级别最高，Ring3最低。Linux使用了Ring3级别运行用户态，Ring0作为内核态。Ring3状态不能访问Ring0的地址空间，包括代码和数据。因此用户态是没有权限去操作内核态的资源的，它只能通过系统调用外完成用户态到内核态的切换，然后在完成相关操作后再有内核态切换回用户态。




 Q：在linux中内核态的权限是最高的，那么在内核态的场景下，操作系统是可以访问任何一个内存区域的，所以操作系统是可以访问到Java堆的这个内存区域的。那为什么操作系统不直接访问Java堆内的内存区域了？
 A：这是因为JNI方法访问的内存区域是一个已经确定了的内存区域地质，那么该内存地址指向的是Java堆内内存的话，那么如果在操作系统正在访问这个内存地址的时候，Java在这个时候进行了GC操作，而GC操作会涉及到数据的移动操作[GC经常会进行先标志在压缩的操作。即，将可回收的空间做标志，然后清空标志位置的内存，然后会进行一个压缩，压缩就会涉及到对象的移动，移动的目的是为了腾出一块更加完整、连续的内存空间，以容纳更大的新对象]，数据的移动会使JNI调用的数据错乱。所以JNI调用的内存是不能进行GC操作的。



Q：如上面所说，JNI调用的内存是不能进行GC操作的，那该如何解决了？
 A：①堆内内存与堆外内存之间数据拷贝的方式(并且在将堆内内存拷贝到堆外内存的过程JVM会保证不会进行GC操作)：比如我们要完成一个从文件中读数据到堆内内存的操作，即FileChannelImpl.read(HeapByteBuffer)。这里实际上File I/O会将数据读到堆外内存中，然后堆外内存再讲数据拷贝到堆内内存，这样我们就读到了文件中的内存。而写操作则反之，我们会将堆内内存的数据线写到对堆外内存中，然后操作系统会将堆外内存的数据写入到文件中。
 ② 直接使用堆外内存，如DirectByteBuffer：这种方式是直接在堆外分配一个内存(即，native memory)来存储数据，程序通过JNI直接将数据读/写到堆外内存中。因为数据直接写入到了堆外内存中，所以这种方式就不会再在JVM管控的堆内再分配内存来存储数据了，也就不存在堆内内存和堆外内存数据拷贝的操作了。这样在进行I/O操作时，只需要将这个堆外内存地址传给JNI的I/O的函数就好了。