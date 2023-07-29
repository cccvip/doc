## GC时机？
### Young Gc什么时候会触发？
新创建的对象在新生代Eden触发进行分配,如果Eden没有足够的空间,则会进行Young Gc清理垃圾

### 什么时候触发Full GC?
- Young GC 之前检查老年代：在要进行 Young GC 的时候，发现老年代可用的连续内存空间 < 新生代历次Young GC后升入老年代的对象总和的平均大小，说明本次 Young GC 后可能升入老年代的对象大小，可能超过了老年代当前可用内存空间,那就会触发 Full GC。
- Young GC 之后老年代空间不足：执行 Young GC 之后有一批对象需要放入老年代，此时老年代就是没有足够的内存空间存放这些对象了，此时必须立即触发一次 Full GC
老年代空间不足，老年代内存使用率过高，达到一定比例，也会触发 Full GC。
- 空间分配担保失败（ Promotion Failure），新生代的 To 区放不下从 Eden 和 From 拷贝过来对象，或者新生代对象 GC 年龄到达阈值需要晋升这两种情况，老年代如果放不下的话都会触发 Full GC。
- 方法区内存空间不足：如果方法区由永久代实现，永久代空间不足 Full GC。
- System.gc()等命令触发：System.gc()、jmap -dump 等命令会触发 full gc。


#### 对象什么时候会进入老年代？
- 长期存活的对象将进入老年代
- 大对象直接进入老年代
- 动态对象年龄判定
- 空间分配担保


#### 能详细说一下 CMS 收集器的垃圾收集过程吗？
- 初始标记（CMS initial mark）
  单线程运行，需要 Stop The World，标记 GC Roots 能直达的对象。
- 并发标记（（CMS concurrent mark）
  无停顿，和用户线程同时运行，从 GC Roots 直达对象开始遍历整个对象图。
- 重新标记（CMS remark）
  多线程运行，需要 Stop The World，标记并发标记阶段产生对象。
- 并发清除（CMS concurrent sweep）
  无停顿，和用户线程同时运行，清理掉标记阶段标记的死亡的对象。

#### G1垃圾收集了解吗？

G1 把连续的 Java 堆划分为多个大小相等的独立区域（Region），每一个 Region 都可以根据需要，扮演新生代的 Eden 空间、Survivor 空间，或者老年代空间。收集器能够对扮演不同角色的 Region 采用不同的策略去处理。
G1 收集器的运行过程大致可划分为以下四个步骤：
- 初始标记（initial mark）
  标记了从 GC Root 开始直接关联可达的对象。STW（Stop the World）执行。
- 并发标记（concurrent marking）
  和用户线程并发执行，从 GC Root 开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象、
- 最终标记（Remark）
  STW，标记再并发标记过程中产生的垃圾。
- 筛选回收（Live Data Counting And Evacuation）
  制定回收计划，选择多个 Region 构成回收集，把回收集中 Region 的存活对象复制到空的 Region 中，再清理掉整个旧 Region 的全部空间。需要 STW。
  

### 线上用的什么垃圾回收器？
> 高吞吐，我们可以回答：因为我们系统是业务相对复杂，但并发并不是非常高，所以希望尽可能的利用处理器资源，出于提高吞吐量的考虑采用Parallel Scavenge + Parallel Old

> Parallel New+CMS的组合，我们比较关注服务的响应速度，所以采用了 CMS 来降低停顿时间。

> 使用G1 垃圾收集器，因为它不仅满足我们低停顿的要求，而且解决了 CMS 的浮动垃圾问题、内存碎片问题。
