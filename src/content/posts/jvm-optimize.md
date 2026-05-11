---
title: JVM调优
published: 2026-05-11
description: JVM调优的核心概念和方法论
tags: [JVM, GC, Performance]
category: Performance
draft: true
---

# JVM调优

JVM调优本质就是在STW、吞吐量、资源占用之间的策略权衡.

调些什么？
内存区域的大小与分配
垃圾回收器的选择与配置
对象晋升与回收策略
线程堆栈与本地内存


啥时候需要调优？
GC 过于频繁： 比如每分钟都有多次 Minor GC。
GC 停顿过长： 单次 Full GC 导致系统卡顿超过 1 秒，影响用户体验。
内存泄漏（Memory Leak）： 内存使用率曲线持续阴跌，无法通过 GC 回收。
吞吐量下降： CPU 大量消耗在 GC 线程上，而不是业务逻辑上。
一定要依赖GC Log来调优

随着JVM的不断发展，调优已经从“人工精细化配置”转向“自适应自动化“。

在JDK8时代
内存区域

-Xms / -Xmx： 建议设为一致，避免 JVM 在业务高峰期因扩容产生性能抖动。

-XX:NewRatio： 默认通常是 2（即老年代是新生代的 2 倍）。如果你的业务是短生命周期的（如大部分 Web 请求），得手动调大新生代，否则对象会很快被赶进老年代。

-XX:SurvivorRatio： 默认是 8。如果 Survivor 区太小，会导致“动态年龄判定”失效，对象直接溢出到老年代。

-XX:MaxMetaspaceSize： 必须设一个上限！因为元空间用的是本地内存，如果不设限制，一旦类加载器泄露，会把整台服务器的物理内存吸干导致 OOM


垃圾回收器：
CMS和Parallel二选一

对象晋升与回收策略：
-XX:MaxTenuringThreshold 默认为 15。如果你的机器内存很大，其实可以调小这个值，让对象早点去老年代，减少新生代复制开销；反之则调大
-XX:PretenureSizeThreshold 避免大对象（如巨型数组）在 Eden 和 Survivor 之间“仰卧起坐”，直接一步到位送入老年代

线程堆栈与本地内存：
-Xss： 默认 1MB。在 Java 8 时代，如果你有数千个线程，光线程栈就会吃掉几个 GB 的物理内存。

局限： 因为这个版本没有“虚拟线程”，你唯一的调优手段就是控制线程池大小，防止线程过多导致 CPU 上下文切换太频繁。

JDK11版本
内存区域：
-Xms / -Xmx： 建议设为一致，避免 JVM 在业务高峰期因扩容产生性能抖动。
还有元空间

垃圾回收：
基本大多数情况选G1，对延迟要求特别低的时候可以选ZGC【不分代】
选G1时需要调整 -XX:MaxGCPauseMillis，控制最大暂停时间

对象晋升：
需要调整Region的大小，-XX:G1HeapRegionSize

线程栈与本地内存：
需要控制线程数量，以及本地内存大小-XX:MaxDirectMemorySize

JDK21版本：
内存：
-Xms / -Xmx，或者容器环境直接用-XX:MaxRAMPercentage=75.0
元空间需要设置上限

垃圾回收器：
基本分代ZGC一家独大

对象晋升与回收：交给JVM

线程与本地内存：
有了虚拟线程以后不用再控制线程数量了，但是需要控制并发数，此外需要注意pin问题

依然需要控制本地内存大小


JDK25版本
JVM参数上类似于JDK21，除了一个新的紧凑头选项，可以节省10%-20%的内存。
GC从ZGC一家独大，变更为分代Shenandoah和分代ZGC双王争霸
此外JVM内部对锁定机制进行了重构，大幅缓解了pin问题（几乎不会有了）
此外ScopedValue用来代替ThreadLocal，来减少内存


