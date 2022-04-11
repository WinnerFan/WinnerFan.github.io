---
layout: post
title: Frequently GC
tags: JVM
---
## Frequently GC

### 基础知识
- 新生代（Eden，Survivor1，Survivor2） Minor GC，老年代 MajorGC。Full GC回收整个堆内存
- 触发条件
    1. System.gc()，使用`-XX:+ DisableExplicitGC`来禁止RMI调用System.gc()
    2. 老年代空间不足，大对象，仍然不足会抛出错误：java.lang.OutOfMemoryError: Java heap space
    3. 永久代满了，未配置CMS GC会Full GC，仍然不足会抛出错误：java.lang.OutOfMemoryError: PermGen space
    4. 老年代CMS GC，GC日志中promotion failed（survivor和老年代都放不下）和concurrent mode failed（老年代放不下）。
    5. Minor GC后进入老年代的平均大小大于老年代的可用内存。

### 操作步骤
1. 查看应用堆内存`jmap -heap <pid>`

```
using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
    MinHeapFreeRatio = 0 //对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小使用比率HeapFreeRatio =(CurrentFreeHeapSize/CurrentTotalHeapSize)小于则扩容 
    MaxHeapFreeRatio = 100 //对应jvm启动参数-XX:MaxHeapFreeRatio设置JVM堆最大可用比率HeapFreeRatio =(CurrentFreeHeapSize/CurrentTotalHeapSize)大于则扩容 
    MaxHeapSize = 2051014656 (1956.0MB) //对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
    NewSize = 42991616 (41.0MB) //对应jvm启动参数-XX:NewSize=设置JVM堆的新生代默认值和最小值
    MaxNewSize = 683671552 (652.0MB) //对应jvm启动参数-XX:MaxNewSize=设置JVM堆的新生代的最大值
    OldSize = 87031808 (83.0MB) //对应jvm启动参数-XX:OldSize=设置JVM堆的老生代的默认大小
    NewRatio = 2 //对应jvm启动参数-XX:NewRatio=:新生代和老生代的大小比率
    SurvivorRatio = 8 //对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden
区与Survivor区的大小比值
    MetaspaceSize = 134217728 (128.0MB) //JVM 元空间的默认值
    CompressedClassSpaceSize = 314572800 (300.0MB)
    MaxMetaspaceSize = 268435456 (256.0MB) //JVM 元空间允许的最大值
    G1HeapRegionSize = 0 (0.0MB) //G1每个Region 空间的大小

Heap Usage:
PS Young Generation
Eden Space:
    capacity = 651689984 (621.5MB) //Eden区总容量
    used = 651689984 (621.5MB) //Eden区已使用
    free = 0 (0.0MB) //Eden区剩余容量
    100.0% used //Eden区使用比率
From Space: //其中一个Survivor区的内存分布
    capacity = 14680064 (14.0MB)
    used = 7143424 (6.8125MB)
    free = 7536640 (7.1875MB)
    48.660714285714285% used
To Space: //另一个Survivor区的内存分布
    capacity = 14155776 (13.5MB)
    used = 0 (0.0MB)
    free = 14155776 (13.5MB)
    0.0% used
PS Old Generation //当前的Old区内存分布
    capacity = 357564416 (341.0MB)
    used = 257587824 (245.65489196777344MB)
    free = 99976592 (95.34510803222656MB)
    72.03955776181039% used

43769 interned Strings occupying 4868088 bytes.
```
堆内存过小，Survivor空间过小，每次GC，Eden容易进入Old，Old区不大，引发Full GC。但是没有自动扩容！

每次垃圾回收后，由MinHeapFreeRatio和MaxHeapFreeRatio决定是否扩缩容，当有空闲比空间时，GC后堆空闲比率不会小于0，不会扩容。

### 性能调优
`-Xmx 4096m -Xms 4096m -Xmn 2048m -XX:-UseAdaptiveSizePolicy`

- 最大内存和堆内存一致，JVM启动直接申请4G内存，避免每次垃圾回收完JVM重新分配内存。
- 新生代和老年代固定为2G（`-XX:NewSize=`新生代初始值和最小值，`-XX:MaxNewSize=`新生代最大值，`-Xmn`新生代初始值、最小值以及最大值。当`-Xmx -Xms`不同时`-Xmn`依然生效），Survivor固定为200M。
- JDK 1.8 默认使用 UseParallelGC 垃圾回收器，该垃圾回收器默认启动了 AdaptiveSizePolicy，（CMS不管怎么设置都为关闭）会动态调整 Eden、Survivor 的大小，有些情况存在Survivor 被自动调为很小，导致老年代增长过快。关闭。

### 其他建议
- `-Xmx -Xms`为老年代的3到4倍，等同于`-XX:MaxHeapSize -XX:InitialHeapSize`
- `-XX:PermSize= -XX:MaxPermSize`应该指定为相同，由于这个值的调整会导致Full GC，应该为稳定后的1.2到1.5
- 新生代空间应该是老年代活动对象大小的1到1.5倍