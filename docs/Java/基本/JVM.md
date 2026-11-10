# 前言

JVM 的堆栈、内存模型这些基础，参考周志明《深入理解 Java 虚拟机》即可。这篇只抓两件事：**GC 怎么选、线上怎么排查**。

# 版本速查（JDK 8 vs JDK 17）

| | JDK 8 | JDK 17 |
|---|---|---|
| 默认收集器 | Parallel Scavenge + Parallel Old | G1 |
| 方法区 | Metaspace（永久代已移除） | Metaspace |
| CMS | 可用 | 已移除（JDK14，JEP 363） |
| G1 | 可用 | 默认 |
| ZGC | 无 | 可用（JDK15 转正，JEP 377） |
| Shenandoah | 无 | 部分发行版可用（JEP 379） |
| GC 日志 | `-XX:+PrintGCDetails` 等 | 统一日志 `-Xlog:gc*` |
| 偏向锁 | 默认开启 | 默认禁用（JEP 374） |

记忆点：**JDK8 默认 Parallel，JDK9 起默认 G1；CMS 在 JDK14 被移除；ZGC 在 JDK17 可用于生产。**

# 内存结构

- 堆：新生代（Eden + S0 + S1）+ 老年代；
- 方法区：JDK8 起是 Metaspace，用本地内存，不再受 `-Xmx` 限制；
- 虚拟机栈 / 本地方法栈 / 程序计数器：线程私有；
- 直接内存：NIO、Netty 常用，受 `-XX:MaxDirectMemorySize` 约束。

# 怎么判断对象死亡

- 引用计数：简单，但解决不了循环引用，JVM 不用；
- **可达性分析**：从 GC Roots 出发，走不到的对象可回收。

常见 GC Roots：虚拟机栈中的局部变量、方法区静态变量、常量引用、JNI 引用、活跃线程、锁监视器等。

# 分代与对象晋升

- 对象优先在 Eden 分配；
- 新生代用复制算法，一次 Minor GC 把存活对象复制到 Survivor；
- 晋升老年代的四条路径：
  1. 年龄达到 `-XX:MaxTenuringThreshold`（默认 15）；
  2. 大对象直接进老年代（`-XX:PretenureSizeThreshold`）；
  3. 动态年龄判定：Survivor 中同年龄对象总和超过一半（`-XX:TargetSurvivorRatio`）；
  4. 分配担保：Survivor 放不下，直接进老年代。

# 常见 GC 算法

| 算法 | 特点 | 问题 |
|---|---|---|
| 标记-清除 | 简单，不移动对象 | 内存碎片、效率不稳定 |
| 标记-复制 | 无碎片，适合存活率低的新生代 | 浪费一半空间、移动成本 |
| 标记-整理 | 无碎片，适合老年代 | 移动对象成本高 |
| 分代收集 | 按对象寿命分区，组合使用 | 实现复杂 |

三色标记（可达性分析的具体实现）：

- 白色：未访问；灰色：已访问但引用未扫完；黑色：已访问且引用扫完。
- 漏标需要同时满足：黑色对象新增指向白色对象的引用，且灰色对象到该白色对象的引用被删除。
- 解决：CMS 用**增量更新**（记下黑→白的新引用），G1 用 **SATB**（记下删除前的旧引用）。

# 常见收集器

## Serial / Serial Old

单线程，STW。适合客户端、小内存、单核。新生代复制，老年代标记-整理。

## ParNew

Serial 的多线程版，主要用来配合 CMS 收集新生代。

## Parallel Scavenge / Parallel Old

吞吐量优先（`吞吐量 = 用户代码时间 / (用户代码时间 + GC 时间)`）。JDK8 默认组合，适合后台批处理。

## CMS（Concurrent Mark Sweep）

作用于老年代，目标是低停顿。流程：

1. 初始标记（STW）：标记 GC Roots 直接可达对象；
2. 并发标记：和用户线程一起遍历对象图；
3. 重新标记（STW）：修正并发标记期间的变化；
4. 并发清除：清理不可达对象。

问题：

- 标记-清除产生**内存碎片**；
- 并发阶段产生**浮动垃圾**；
- 并发失败（Concurrent Mode Failure）会退化成 Serial Old，停顿变长；
- 对 CPU 敏感。

常用参数：`-XX:CMSInitiatingOccupancyFraction=92`、`-XX:+UseCMSCompactAtFullCollection`、`-XX:CMSFullGCsBeforeCompaction`。

## G1（Garbage First）

把堆划成大小相等的 Region（默认约 2048 块），每个 Region 运行时动态扮演 Eden、Survivor、Old 或 Humongous。逻辑上仍分代，但物理上不连续。

- 年轻代用复制，整体回收可看作复制 + 整理；
- 维护 Region 的回收价值（垃圾占比）优先级队列，在 `-XX:MaxGCPauseMillis`（默认 200ms）约束下优先回收收益最高的 Region；
- 用 Remembered Set 记录跨 Region 引用，避免全堆扫描；
- 并发标记用 SATB，最终标记和筛选回收需要 STW。

关键参数：`-XX:MaxGCPauseMillis`、`-XX:G1HeapRegionSize`、`-XX:InitiatingHeapOccupancyPercent`（默认 45）。

## ZGC

JDK11 引入、JDK15 转正、JDK17 可用于生产。目标停顿 **10ms 以内**，支持 TB 级堆。

核心：**染色指针（Colored Pointers）+ 读屏障（Load Barrier）**，把标记、转移、重定位都做成并发，STW 只做根扫描等极短工作。JDK17 的 ZGC 不分代，JDK21 起有分代 ZGC。

## 其他

- Shenandoah：Red Hat 贡献的低停顿收集器，部分 OpenJDK 发行版可用；
- Epsilon：不做任何回收，只分配，用于性能测试。

## 怎么选

- 吞吐优先、后台任务：Parallel；
- 低延迟、堆 4G~几十 G：G1（JDK8 也能用）；
- 超大堆、极致低停顿：ZGC（JDK17+）；
- 老项目 JDK8 且停顿敏感：G1 或 CMS（能升级就优先 G1）。

# 八股速查

## 1. 如何判断对象可以被回收？

可达性分析：从 GC Roots 出发不可达的对象，且经过一次标记/清理仍不可达，才真正回收（还要看 `finalize`，但不建议依赖）。

## 2. Minor GC、Major GC、Full GC 的区别？

- Minor / Young GC：只回收新生代，频繁、停顿短；
- Major / Old GC：只回收老年代（CMS 的并发清除属于此类）；
- Full GC：整堆 + 方法区，停顿最长。

## 3. 什么时候触发 Full GC？

- 老年代空间不足；
- 元空间不足；
- 分配担保失败；
- `System.gc()`；
- CMS 并发失败退化成 Serial Old。

## 4. CMS 和 G1 的区别？

| | CMS | G1 |
|---|---|---|
| 范围 | 老年代 | 整堆（Region） |
| 算法 | 标记-清除 | 复制 + 整理 |
| 碎片 | 有 | 基本没有 |
| 停顿 | 不可预测 | 可设目标 |
| 大堆表现 | 差 | 好 |

## 5. G1 怎么做到可预测停顿？

不是固定回收多少，而是**在 `MaxGCPauseMillis` 约束下，优先回收垃圾最多、成本最低的 Region**，用收益模型挑一批 Region 组成回收集。

## 6. ZGC 为什么停顿这么短？

染色指针把对象状态编码进指针，配合读屏障，在应用线程运行时并发完成标记和对象转移；STW 阶段只做根扫描，时间与堆大小基本无关。

## 7. 什么是 STW、SafePoint、Safe Region？

- STW：Stop The World，GC 时暂停所有应用线程；
- SafePoint：线程可以安全暂停的位置（循环末尾、方法返回前、调用返回后、异常抛出点）；
- Safe Region：引用关系不会变化的代码区域，线程在里面时 GC 不必等它走到 SafePoint。

## 8. 内存泄漏和内存溢出有什么区别？

- 内存泄漏：对象该回收却回收不掉，老年代持续上涨，最终 OOM；
- 内存溢出：内存确实不够用，可能是一次性申请过大对象。

## 9. 常见 OOM 有哪些？

`Java heap space`、`GC overhead limit exceeded`、`Metaspace`、`Direct buffer memory`、`unable to create new native thread`、`Requested array size exceeds VM limit`。

## 10. JDK8 和 JDK17 默认 GC 分别是什么？

JDK8 默认 Parallel Scavenge + Parallel Old；JDK17 默认 G1。

# 分析思路（重点）

## 排查总流程

1. **确认现象**：接口超时？CPU 高？内存涨？OOM？
2. **定位进程**：`jps -l` / `ps -ef | grep java`。
3. **看 GC**：`jstat -gcutil` + GC 日志，判断是 YGC 频繁、FGC 频繁，还是单次 STW 过长。
4. **看堆**：`jmap -histo:live` 看对象分布，必要时 dump。
5. **看线程**：`jstack` + `top -Hp`，判断是 GC 线程还是业务线程占 CPU。
6. **分析 dump**：MAT 的 Leak Suspects、Dominator Tree、Path to GC Roots。
7. **定位代码**：结合业务日志和改动时间，找到分配点 / 泄漏点。
8. **修复并验证**：改代码或调参，复测 GC 指标。

## 命令速查

```bash
# 进程
jps -l

# GC 概览（每 1 秒一次，共 10 次）
jstat -gcutil <pid> 1000 10

# 堆信息 / 启动参数
jcmd <pid> GC.heap_info
jcmd <pid> VM.flags

# 对象直方图（:live 会先触发一次 Full GC）
jmap -histo:live <pid> | head -30

# 堆转储
jmap -dump:live,format=b,file=heap.hprof <pid>

# 线程栈
jstack -l <pid> > stack.txt

# 找最耗 CPU 的线程
top -Hp <pid>
printf "%x\n" <tid>
jstack <pid> | grep -A 20 <十六进制tid>
```

GC 日志：

```bash
# JDK8
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log

# JDK9+
-Xlog:gc*,gc+heap=debug:file=gc.log:time,uptime,level,tags
```

生产必备：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
```

## 判读技巧

- **Full GC 后老年代明显回落**：多半不是泄漏，是分配速率高或堆偏小；
- **Full GC 后老年代不降**：泄漏，去 MAT 看谁在强引用；
- **YGC 频繁、每次回收很少**：新生代偏小或对象创建太快；
- **FGC 频繁**：老年代增长快、元空间不足，或有人调了 `System.gc()`；
- **单次 STW 很长**：堆大 + 单线程收集器，或一次晋升对象太多；
- **CPU 高但 GC 占比低**：看业务线程热点（`jstack`、`arthas thread -n 3`）；
- **线程数持续上涨**：线程池 / 连接池泄漏；
- **Metaspace OOM**：动态生成类过多（CGLIB、反射、热部署、脚本引擎）。

## 实战案例

**案例一：Full GC 10 多秒**

线上接口大面积超时，young gc 频繁，full gc 单次 10 多秒。`jmap -dump` 后用 MAT 发现某对象异常膨胀，定位到 fastjson 的一个已知 issue，临时用 Jackson 替换后恢复。

**案例二：内存持续上涨**

程序大量批插入且无法缓存，Druid 内部 map 越来越大。解决思路是控制批处理的数据量 / 及时清理，而不是一味加内存。

**案例三：CPU 飙高**

线程池创建了大量线程且处于 hanging，既不释放也不关闭。用 `top -Hp` + `jstack` 定位到线程栈后修复。

> 更详细的过程见 [内存调优.md](内存调优.md)。

# 参数速查

| 参数 | 作用 |
|---|---|
| `-Xms` / `-Xmx` | 堆初始 / 最大，建议设成一样避免动态扩缩 |
| `-Xmn` / `-XX:NewRatio` | 新生代大小 / 新老比例 |
| `-XX:SurvivorRatio` | Eden 与单个 Survivor 的比例 |
| `-XX:MaxTenuringThreshold` | 晋升年龄阈值，默认 15 |
| `-XX:MetaspaceSize` / `MaxMetaspaceSize` | 元空间初始 / 上限 |
| `-XX:+UseG1GC` | 使用 G1 |
| `-XX:MaxGCPauseMillis` | G1 目标停顿，默认 200ms |
| `-XX:G1HeapRegionSize` | G1 Region 大小 |
| `-XX:+UseZGC` | 使用 ZGC（JDK17+） |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时自动 dump |
| `-Xlog:gc*` | JDK9+ 统一 GC 日志 |

# 参考

- 周志明《深入理解 Java 虚拟机》
- JEP 363（移除 CMS）、JEP 374（禁用偏向锁）、JEP 377（ZGC 转正）、JEP 379（Shenandoah 转正）
- [内存调优.md](内存调优.md)、[内存调优第二篇.md](内存调优第二篇.md)
