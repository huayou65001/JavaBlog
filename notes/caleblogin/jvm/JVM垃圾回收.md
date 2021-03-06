# JVM垃圾回收

- 如何判断对象是否死亡（两种方法）
- 简单的介绍一下强引用、软引用、弱引用、虚引用（虚引用与软引用和弱引用的区别、使用软引用能带来的好处）
- 如何判断一个常量是废弃常量
- 如何判断一个类是无用的类
- 垃圾收集有哪些算法，各自的特点？
- HotSpot为什么要分为新生代和老年代？
- 常见的垃圾回收器有哪些？
- 介绍一下CMS,G1收集器
- Minor Gc 和 Full GC 有什么不同呢？

## 垃圾回收器

### 为什么分代？
优化GC性能

### CMS
- 以获取最短的停顿时间为目标，基于标记-清除算法，分为初始标记，并发标记，重新标记，并发清除。初始标记和重新标记需要STW(Stop the world)，初始标记为了标记GC Roots能直接关联的对象，速度很快。并发标记从GC Roots的直接关联对象遍历整个对象图，耗时较长但不需要停顿用户线程。并发标记是为了重新标记用户线程执行而产生变化的那部分对象。并发清除，清除已死亡的对象，不需要移动存活对象。
- 对处理器资源敏感，并发阶段虽然不会导致用户线程暂停，但是会降低吞吐量。无法处理浮动垃圾，可能出现并发失败而导致Full GC。采用标记-清除算法，会产生碎片。

### G1
- 堆被分为新生代和老年代，G1之前的垃圾回收器要么对新生代进行垃圾回收，要么对老年代进行垃圾回收，G1对新生代和老年代同时进行垃圾回收，选取价值最大的Region进行回收。哪块存放的垃圾最多，回收效益最大就回收哪一块。
- 跟踪各Region垃圾的价值，价值即回收垃圾后所获得的空间大小及回收所需要时间的经验值。后台维护一个优先级列表，每次根据用户设定的允许的垃圾回收时间优先处理获得价值最大的Region。保证了G1在有限的时间内获得尽可能高的回收效率。
- 分为4个阶段：初始标记，并发标记，重新标记，筛选回收。初始标记仅仅标记GC Roots直接关联的对象，让下一阶段用户线程并发运行时，能正确的在Region内分配内存。需要STW但耗时较短。并发标记，耗时较长，对堆中的对象做可达性分析，扫描整个堆的对象图，可与用户线程并发执行。重新标记：重新标记用户线程运行期间产生变化的那部分记录。筛选回收：按照Region的成本和价值进行排序，按照用户期望的停顿时间来制定回收计划。停顿用户线程，多条回收线程并发执行。
- 整体上基于标记-整理的算法，局部是基于复制算法，运行期间不会产生内存空间碎片。可预测的停顿时间。


