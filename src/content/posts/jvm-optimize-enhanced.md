---
title: JVM调优指南
published: 2026-05-11
description: JVM调优的诊断框架、各版本特点、工具使用、实战案例，从JDK8到JDK25的完整演进。
tags: [JVM, GC, Performance, Java]
category: Performance
draft: false
---

# JVM调优

JVM调优本质就是在 **STW（Stop The World）、吞吐量、资源占用** 这三者之间进行策略权衡。没有完美的配置，只有最适合当前业务的配置。

## 调优的诊断框架

在开始调优前，你需要建立一个闭环的诊断→调优→验证流程：

```
收集指标 → 识别瓶颈 → 针对性调优 → 对比验证 → (循环)
```

1. **收集指标** - 启用 GC 日志，持续监控
2. **识别瓶颈** - 根据日志分析具体问题（GC 频率过高？单次暂停过长？内存持续上升？）
3. **针对性调优** - 根据业务特点调整对应参数
4. **对比验证** - 同一指标的调优前后数据对比

**重要：无 GC 日志的调优都是瞎蒙。** 这是我反复强调的，因为很多人凭感觉调参，结果越调越糟。

## 调些什么？

JVM 调优的主要维度：

- **内存区域的大小与分配** - 决定何时触发 GC，影响 GC 频率
- **垃圾回收器的选择与配置** - 决定 GC 的执行方式和效率  
- **对象晋升与回收策略** - 决定对象在各代的流动，影响 GC 复杂度
- **线程堆栈与本地内存** - 决定能支持多少并发，影响资源占用

## 啥时候需要调优？

出现以下情况，说明确实需要调优：

- **GC 过于频繁** - 比如每分钟都有多次 Minor GC，业务线程几乎没机会跑完整逻辑
- **GC 停顿过长** - 单次 Full GC 导致系统卡顿超过 1 秒，用户能感觉到响应变慢
- **内存泄漏** - 内存使用率曲线持续上升，无法通过 GC 回收
- **吞吐量下降** - CPU 大量消耗在 GC 线程上，而不是业务处理上

## GC 日志的采集和诊断

要诊断问题，必须先启用 GC 日志。在 JVM 启动参数中加入：

```
-XX:+PrintGCDetails \
-XX:+PrintGCDateStamps \
-XX:+PrintGCApplicationStoppedTime \
-Xloggc:/var/log/gc.log \
-XX:+UseGCLogFileRotation \
-XX:NumberOfGCLogFiles=5 \
-XX:GCLogFileSize=100M
```

采集到日志后，用这些工具诊断：

**jstat** - 实时看 GC 统计
```bash
jstat -gc <pid> 1000  # 每秒输出一次，看各代大小和 GC 次数
```

**jmap** - 定位对象泄漏
```bash
jmap -dump:live,format=b,file=heap.bin <pid>  # dump 堆快照用工具分析
```

**jvisualvm** - 图形化工具，能直观看 GC 曲线、内存占用、线程状态

**Spring Boot 项目** - 直接用 `/actuator/metrics/jvm.gc.*` 或接入 Prometheus + Grafana

## 版本演进：从手工到自适应

随着 JVM 发展，调优从"人工精细化配置"转向"自适应自动化"。不同 JDK 版本的调优重点差异很大。

---

## JDK 8 时代

这个时代的特点：**参数众多，需要手工精细化配置**。

### 内存区域

**`-Xms / -Xmx`** - 建议设为一致，避免 JVM 在业务高峰期因动态扩容产生性能抖动。
```
-Xms4G -Xmx4G
```

**`-XX:NewRatio`** - 新生代与老年代大小比例，默认是 2（老年代是新生代的 2 倍）
- 如果业务是短生命周期的（如大部分 Web 请求），应该调大新生代，否则对象会很快被赶进老年代，导致 Full GC 频繁
- 业务特点不同，调整值不同，必须通过 GC 日志验证

**`-XX:SurvivorRatio`** - Eden 与 Survivor 大小比例，默认是 8（Eden:Survivor0:Survivor1 = 8:1:1）
- Survivor 区太小会导致"动态年龄判定"失效，对象直接溢出到老年代
- 可以根据 GC 日志中对象在 Survivor 区的存活情况调整

**`-XX:MaxMetaspaceSize`** - 元空间上限，必须设！
```
-XX:MaxMetaspaceSize=256M
```
因为元空间用的是本地内存（JDK 8 开始用元空间替代永久代）。如果不设限制，一旦类加载器泄漏（比如动态生成字节码的框架如 CGLIB），会把整台服务器的物理内存吸干导致 OOM。

### 垃圾回收器：CMS 和 Parallel 二选一

**Parallel GC**（吞吐量优先）：
```
-XX:+UseParallelGC -XX:+UseParallelOldGC
```
- 适用于 **批处理、离线任务**，不关心单次暂停时间，只要总吞吐量高
- 多线程并行 GC，充分利用多核

**CMS GC**（延迟优先）：
```
-XX:+UseConcMarkSweepGC
```
- 适用于 **交互式应用、Web 服务**，对用户响应延迟敏感
- 大部分时间与业务线程并发 GC，减少 STW 时间

> **CMS 已在 JDK 9 被标记为废弃，JDK 14 完全移除**，现在不建议新项目用 JDK 8，但维护老系统还能见到。

### 对象晋升与回收策略

**`-XX:MaxTenuringThreshold`** - 对象晋升到老年代的年龄阈值，默认是 15
- 如果机器内存很大，可以调小这个值，让对象早点进老年代，减少新生代复制开销
- 反之，内存紧张可以调大，让对象在新生代多存活几次，减少老年代压力

**`-XX:PretenureSizeThreshold`** - 超过这个大小的对象直接在老年代分配，避免大对象在 Eden 和 Survivor 间"仰卧起坐"
```
-XX:PretenureSizeThreshold=1M
```

### 线程堆栈与本地内存

**`-Xss`** - 单个线程栈大小，默认 1MB
```
-Xss512K
```
在 JDK 8，如果有数千个线程，光线程栈就会吃掉几个 GB 的物理内存。

**局限**：没有虚拟线程的年代，唯一的调优手段就是控制线程池大小，防止线程过多导致 CPU 上下文切换太频繁。

---

## JDK 11 版本

这个时代的特点：**G1 GC 成熟，开始默认使用，参数复杂度降低**。

### 内存区域

**`-Xms / -Xmx`** - 同 JDK 8，建议一致避免动态扩容。

**元空间** - 同样需要设上限 `-XX:MaxMetaspaceSize`。

### 垃圾回收器：G1 当道

**基本结论**：大多数情况选 G1，对延迟要求特别低的时候可以选 ZGC。

```
-XX:+UseG1GC
```

G1 的核心特点：**分代且分区**。将堆分成若干个 Region，既有新生代/老年代概念，又能灵活调度。

**`-XX:MaxGCPauseMillis`** - 控制 G1 的最大暂停时间，默认 200ms
```
-XX:MaxGCPauseMillis=100
```
- 设置过小（如 50ms）会导致 GC 频率增加，因为 G1 需要更频繁地做 GC 来保证暂停时间
- 设置过大（如 500ms）会让单次暂停变长，影响用户体验
- 通常在 50-200ms 之间调整，根据业务对响应时间的要求

### 对象晋升

**`-XX:G1HeapRegionSize`** - G1 的 Region 大小，通常自动计算，无需手工设置
- 如果必须调整，可以根据 GC 日志中 Region 的 occupancy 情况手工指定

### 线程栈与本地内存

同 JDK 8，需要控制线程数量。

**`-XX:MaxDirectMemorySize`** - 堆外内存上限，重要！
```
-XX:MaxDirectMemorySize=1G
```
防止 DirectByteBuffer、Unsafe 等导致的堆外内存泄漏把机器吃掉。

---

## JDK 17+ 版本（包括 JDK 21）

这个时代的特点：**垃圾回收器自动化，虚拟线程出现，参数精简到只需要调少数几个**。

### 内存区域

**容器环境推荐用百分比设置：**
```
-XX:MaxRAMPercentage=75.0
```
这样 JVM 能自动感知容器的 cgroup 限制，不需要手工计算。

**元空间上限依然要设**：
```
-XX:MaxMetaspaceSize=256M
```

### 垃圾回收器：ZGC 一家独大

**基本结论**：选 ZGC（分代 ZGC）。

```
-XX:+UseZGC
```

ZGC 的特点：
- **极低延迟** - 单次暂停通常在毫秒级别，远低于 G1
- **无需调参** - 基本不需要像 G1 那样调 MaxGCPauseMillis
- **适合所有场景** - 既适合高吞吐的后台系统，也适合低延迟的交互应用

**使用场景：** 如果系统对延迟敏感（如实时交易系统、即时通讯），ZGC 是首选。

### 对象晋升与回收

交给 JVM 自动判断，基本不需要手工调优。

### 线程与本地内存

**虚拟线程的出现改变了一切：**
```
// JDK 21+
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```
不再需要控制线程池大小，因为虚拟线程轻量级，可以创建百万级别。

**但需要注意 Pin 问题：**
- 当虚拟线程执行 `synchronized` 方法、调用 native 方法、或进行 JNI 操作时，会被 Pin 到平台线程
- 被 Pin 的虚拟线程无法切换，影响调度效率，多个 Pin 线程会导致平台线程阻塞
- **解决方案**：用 `ReentrantLock` 替代 `synchronized`，避免 native 调用，或升级到最新的 JDK 版本（JDK 25+ 大幅缓解了这个问题）

**本地内存依然要控制：**
```
-XX:MaxDirectMemorySize=2G
```

---

## JDK 25 版本

这个时代的特点：**GC 多选，Pin 问题基本解决，内存压缩**。

### 内存区域

同 JDK 21，参数精简。

**新特性：紧凑头（Compact Object Headers）**
```
-XX:+UseCompactObjectHeaders
```
- 对象头从 128 位压缩到 64 位，能节省 10%-20% 的内存
- 适合内存紧张的场景

### 垃圾回收器：双王争霸

**分代 Shenandoah** 和 **分代 ZGC** 开始竞争：

```
-XX:+UseShenandoahGC  # 分代 Shenandoah
-XX:+UseZGC           # 分代 ZGC
```

两者都是低延迟 GC，性能接近，选哪个看个人偏好。Shenandoah 由 Red Hat 维护，ZGC 由 Oracle 维护。

### 线程模型改进：Pin 问题几乎消失

JVM 内部对锁定机制进行了重构，虚拟线程被 Pin 的情况大幅减少，基本不需要担心 Pin 导致的性能问题。

**新 API：ScopedValue 替代 ThreadLocal**
```java
private static final ScopedValue<String> USER = ScopedValue.newInstance();

// 使用
ScopedValue.where(USER, "alice").run(() -> {
    System.out.println(USER.get());  // alice
});
```
- ScopedValue 的生命周期更短，作用域更明确
- 减少虚拟线程因 ThreadLocal 导致的内存占用
- 在高并发场景下性能更好

---

## 实战调优案例

### 场景 1：Web 服务，GC 频繁

**现象：** 每秒都有 Minor GC，响应延迟增加。

**诊断步骤：**
1. 看 GC 日志，观察新生代占用率
2. 用 jstat 看各代的升级频率
3. 用 jmap dump 堆，检查是否有大量短生命对象

**可能的调优方案：**
- 调大新生代（-XX:NewRatio）
- 增加 Survivor 大小（-XX:SurvivorRatio）
- 优化业务代码，减少临时对象创建

### 场景 2：内存持续上升，疑似泄漏

**诊断步骤：**
1. 看 GC 日志，Old 区是否在持续增长
2. 采集多个堆快照（间隔 1 小时），用 MAT 或 YourKit 对比
3. 找出两个快照间新增的对象

**可能的根因：**
- 业务代码的静态集合（Map、List）不清理
- 类加载器泄漏（很多框架的 Spring Bean 缓存）
- 第三方库的缓存没有过期策略

---

## 总结：调优的黄金法则

1. **不看日志不调参** - 感觉不准，数据说话
2. **调优前对比基准** - 知道调优前是什么样子
3. **一次只改一个参数** - 改多了看不出效果
4. **定期监控验证** - 调完不等于调好，要持续看
5. **相信 JVM 的自适应** - 现代 JVM 版本已经很聪明，手工调优的收益递减
6. **优先优化代码** - 比调 JVM 参数收益更大
