# JVM 调优实战：K8s 资源监听系统内存优化

> 项目：`lb-gnc-k8s-dashboard`  规格：4C8G 容器  JDK：17  GC：G1

---

## 一、项目背景

`lb-gnc-k8s-dashboard` 是公司内部 GNC 平台的 **K8s 资源监听同步服务**，核心职责是：

- 通过 **Kubernetes Informer 机制** Watch 多个集群的 Namespace、RoleBinding、Service CRD 资源变更；
- 将变更数据实时翻译并落库到 MariaDB，同时刷新 Redis 鉴权缓存，为上层控制台 `lb-sphere` 提供权限数据。

随着接入集群数量从 **3 个** 增长到 **10+ 个**，监听资源总量突破 **10 万**，系统出现严重的内存和 GC 问题。

---

## 二、为什么 K8s 监听会导致内存暴涨

理解根因是调优的前提。Informer 机制的工作模式决定了它天然是"内存大户"：

1. **List-Watch 初始化**：启动时先执行一次全量 List，把所有监听资源对象完整加载进 JVM 堆内存维护本地缓存（Local Store），以便后续 Watch 事件能实时对比并回调。
2. **资源对象本身很重**：每个 K8s 资源对象包含大量 Labels、Annotations、Status 等元数据，10 万资源对象在内存中可能轻松占用数 GB。
3. **事件频率比预期高**：Pod 状态变化、滚动发布等都会触发 Watch 事件，大型微服务环境下每天可能产生数十万次事件，**事件处理任务堆积在线程池队列中进一步放大内存压力**。
4. **本地缓存无上限**：代码中使用无界 HashMap 缓存已处理资源的标识（`hasBeenProcessed` Map），随着处理量增加无限膨胀，既不淘汰也不清理。

---

## 三、问题发现：告警触发

我们基于 **Prometheus + Grafana + AlertManager** 建立了完善的监控告警体系。某天凌晨收到如下一系列告警：

| 告警类型 | 现象 |
|---------|------|
| **内存告警** | 容器内存使用率 > 90%，持续 15 分钟；老年代占用率升至 85% |
| **GC 告警** | 30 分钟内 Full GC 发生 5 次；单次 GC 暂停 > 500ms；GC 后内存回收 < 20% |
| **性能告警** | API 响应延迟 > 1s；线程池任务拒绝数激增 |

**登录监控平台后进一步确认：**
- 内存呈**阶梯式上升**，每次 GC 后回落幅度极小 → 典型内存泄漏/无限增长特征
- 线程池队列长度持续攀升不下降 → 生产速度 > 消费速度，任务严重积压
- GC 线程 CPU 占比异常（> 15%），GC 本身消耗了大量 CPU

**关键 GC 日志片段（JDK 17 G1）：**
```
[2025-09-18T02:15:26] GC(325) Pause Young (G1 Evacuation)  3072M->2867M(4096M)  756ms
[2025-09-18T02:25:42] GC(342) Pause Full  (G1 Compaction)  3890M->2356M(4096M) 4832ms
```

Full GC 暂停近 **5 秒**，且回收的空间仅 1.5 GB，说明堆里存在大量无法被 GC 的强引用对象。

---

## 四、问题根因定位

### 工具链
- **平台监控大盘**（PROC 线程数、ThreadPool 队列长度、GC 次数/暂停时间）
- **FullGC 自动触发 Heap Dump + Thread Dump**（dump 文件自动上传 OSS）
- **JProfiler** 分析 Heap Dump，找出大对象和内存引用链

### 定位结论

| 问题 | 根因 |
|------|------|
| **内存无限增长** | `userMipMap`（用于去重的 HashMap）使用无界 Map，随处理量线性增长，从不淘汰 |
| **线程池队列积压** | 三个 Listener（ServiceListener、RoleBindingListener、NamespaceListener）均使用大容量无界 `LinkedBlockingQueue`，大量未消费任务对象堆积在队列中占用堆内存 |
| **单次处理效率低** | 每条 K8s 事件单独开一次数据库事务写入，高频事件下 DB 连接池耗尽，处理延迟反向加剧队列积压 |

### 话术总结*

改平台的权限资源获取服务在某段时间内，连续收到内存告警：容器内存使用率超过 90%，监控上内存曲线呈阶梯式上升，Full GC频繁且每次 Full GC 后回落幅度极小，且 GC 停顿时间较长（>500ms），说明堆里存在大量无法被回收的强引用对象。同期线程池队列长度指标也在持续攀升，说明消费速度已经跟不上事件产生速度。

鉴于指标异常但不确定是哪些对象占用了内存，我在非高峰期手动触发了一次 Heap Dump（Heap Dump 会有短暂 STW，对性能有影响，所以不常开），用 JProfiler 分析 dump 文件，

- 一是去重用的 HashMap 无界增长从不淘汰
- 二是三个 K8s 监听器的线程池使用了大容量无界队列 `LinkedBlockingQueue`，初始化阶段数万个事件对象积压在队列中无法及时回收
  
这就是内存持续上涨且 GC 回收效率极低的根本原因。

---

## 五、分阶段优化方案

### 阶段一：线程池优化——用「有界队列」切断内存无限增长

**问题**：`LinkedBlockingQueue` 容量 500，但实际相当于无限（初始化大量事件时 500 瞬间被打满，触发扩展），堆积任务对象是内存增长的直接原因。

```java
// 优化前：无界队列，线程数过多
private final ExecutorService namespaceExecutor = new ThreadPoolExecutor(
    5, 30, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(500),              // 实际表现为无界
    new ThreadPoolExecutor.CallerRunsPolicy()
);

// 优化后：有界队列 + 精简线程数
private final ExecutorService namespaceExecutor = new ThreadPoolExecutor(
    2, 5, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),               // 严格有界，队满触发 CallerRunsPolicy
    new ThreadPoolExecutor.CallerRunsPolicy()    // 调用方自己执行，实现背压，不丢任务   
);
```

**为什么减少线程数？**
- K8s Informer 事件的处理主要瓶颈在 DB 写入 I/O，线程数不是越多越好；
- 过多线程 → 频繁上下文切换 → 增加 CPU 消耗和每个线程的栈内存占用。

**效果**：线程数降低 80%，上下文切换次数降低 65%，内存占用下降约 15%。

---

### 阶段二：批处理机制——合并 DB 写入，降低事务开销

**问题**：初始化时 List 操作会产生大量 Add 事件（日志可见 `ServiceListener Add count: 33000+`），每个事件单独落库，数据库连接饱和，处理速度跟不上入队速度。

```java
// 批处理消费线程（以 NamespaceListener 为例）
private final BlockingQueue<V1Namespace> batchQueue = new ArrayBlockingQueue<>(100);

@PostConstruct
public void initBatchProcessor() {
    Thread batchThread = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                List<V1Namespace> batch = new ArrayList<>(20);
                V1Namespace first = batchQueue.poll(100, TimeUnit.MILLISECONDS);
                if (first != null) {
                    batch.add(first);
                    batchQueue.drainTo(batch, 19);   // 最多批量取 20 条
                }
                if (!batch.isEmpty()) {
                    processBatch(batch);              // 单次事务批量写入
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }, "namespace-batch-processor");
    batchThread.setDaemon(true);
    batchThread.start();
}
```

**效果**：DB 写入次数降低约 90%，CPU 使用率降低 25%，内存占用进一步下降约 20%。

---

### 阶段三：缓存管理优化——给无界 Map 加"天花板"

**问题**：`hasBeenProcessed` 检查使用普通 `HashMap`，Key 永不删除、无上限。

```java
// 优化前：普通 HashMap，无限增长
private final Map<String, String> processedMap = new HashMap<>();

// 优化后：LRU 有界 Map，超过 1000 条自动淘汰最旧的
private final Map<String, String> processedMap = Collections.synchronizedMap(
    new LinkedHashMap<String, String>(1000, 0.75f, true) {
        @Override
        protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
            return size() > 1000;
        }
    }
);
```

**效果**：内存占用稳定不再增长；Full GC 频率从**每小时 5~6 次**降至**每天 1~2 次**；GC 暂停时间减少 80%。

---

### 阶段四：JVM 参数调优

容器规格 **4C8G**，JVM 参数优化如下：

```bash
# 堆内存：留出非堆空间（Metaspace、直接内存、线程栈等约 1~1.5G）
-Xms2048m -Xmx2560m

# G1 GC 核心参数
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # 目标最大暂停时间 200ms
-XX:InitiatingHeapOccupancyPercent=45  # 堆使用率 45% 时触发并发 GC，提前回收防止堆满
-XX:G1HeapRegionSize=4m            # 单个 Region 大小（堆 2560m 时建议 4m）
-XX:ConcGCThreads=2                # 并发 GC 线程数（I/O 密集型服务避免占用业务线程）
-XX:ParallelGCThreads=4            # STW 阶段并行线程数，与容器核数匹配

# GC 日志（轮转，避免日志撑满磁盘）
-Xlog:gc*=info:file=/apps/logs/gc-%t.log:time,uptime,tags:filecount=5,filesize=20m

# OOM 时自动生成 Heap Dump（配合监控平台自动上传 OSS）
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/apps/logs/
```

**为什么选 G1 而不是 Parallel GC？**

| 指标 | G1 GC（选用） | Parallel GC（实验） |
|------|--------------|-----------------|
| 吞吐量 | 92% | 96%（+4%） |
| 平均暂停时间 | 120ms ✅ | 350ms |
| 最大暂停时间 | 350ms ✅ | 850ms |
| CPU 使用率 | 25% | 22% |

Parallel GC 吞吐量高 4% 但最大暂停时间高出 **2.4 倍**。本系统需要实时响应 K8s 事件并为上层鉴权接口提供服务，延迟比吞吐量更重要，因此选用 G1。

---

## 六、优化总体效果

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| 容器内存使用率 | 峰值 90%+ | 稳定 50~60% |
| CPU 使用率 | 峰值 45% | 25% 左右 |
| Full GC 频率 | 每小时 5~6 次 | 每天 1~2 次 |
| GC 最大暂停时间 | 4.8s | < 350ms |
| 单节点处理能力 | ~5 万资源 | 10 万+ 资源 |
| 系统连续稳定运行 | 每周多次告警 | 数月无告警 |

> 后续还对容器规格做了扩容（8G → 12G），为后续业务增长提供余量，避免再次触发内存告警。

---

## 七、面试话术（速查版）

### 背景介绍（30 秒）

> 我们有一个内部 K8s 资源监听同步服务，通过 Kubernetes Informer 机制实时 Watch 10+ 集群的 Namespace、RoleBinding、Service CRD 资源变更，并将变更数据翻译落库到 MariaDB，供上层控制台做接口鉴权。随着接入集群从 3 个增长到 10+ 个，监听资源总量突破 10 万，系统出现严重的内存占用和 GC 问题。

### 问题发现（30 秒）
> 某段时间内持续收到告警：容器内存使用率超过 90%，且内存曲线呈阶梯式上升，Full GC 频繁，每次 Full GC 后回落幅度极小，且 GC 停顿时间很长（秒级）；同期线程池队列长度指标也在持续攀升。这是一个强信号——堆里有大量被强引用持有、GC 动不了的对象。

> 我在业务低峰期，开启 FullGC 自动触发的 Heap Dump 文件，用 JProfiler 分析，定位到几个根因：一是去重 HashMap 无界增长；二是线程池使用大容量无界队列，初始化时 33000+ 个事件对象堆积在队列导致内存爆炸。

> 详细说明：于是我在业务低峰期手动触发了一次 Heap Dump（Heap Dump 会有短暂 STW，不常开），用 JProfiler 分析：按内存占用降序看对象列表，两个异常直接跳出来：一是 HashMap$Node 实例数超过百万，引用链追溯到一个 Spring 单例 Bean 的成员字段，生命周期与应用共存，永远不会被 GC；二是大量 V1RoleBinding K8s 对象被线程池的 workQueue 强引用，同时 Thread 视图显示线程基本全 BLOCKED 在数据库连接等待，消费速度极慢导致 27000+ 个任务堆积在队列，这就是内存无法回收的根本原因。

> 系统接入了公司基于 Prometheus + Grafana 搭建的 JVM 监控平台，通过 JMX Exporter 覆盖了堆内存使用率、老年代使用率、GC 次数和停顿时间等标准指标，同时通过 Actuator 埋点监控了线程池队列长度。


### 解决方案（45 秒）

> 我按三个层面依次优化。

> 第一，代码层面：把去重用的无界 HashMap 改为 LRU 有界 Map（容量上限 1000，超出自动淘汰最旧条目）；把三个监听器线程池的无界队列改为有界 ArrayBlockingQueue（容量 100），并下调线程数——因为这类 I/O 密集型场景，线程数过多反而加剧上下文切换，CallerRunsPolicy 拒绝策略也保证了任务不丢失；同时引入批处理机制，将单条 DB 写入合并为每批 20 条，减少约 90% 的数据库事务次数。

> 第二，代码层面优化完之后，系统内存压力明显下降，但 GC 表现还有优化空间，于是进一步做了 JVM 参数调优。
> 原来的问题：没有显式配置堆内存，JVM 默认按物理内存 1/4 动态调整，4C8G 容器下最大堆约 2G，而且 -Xms 和 -Xmx 不一致，JVM 会在运行中频繁扩缩堆，扩缩本身就会额外触发 GC，加剧了停顿问题。
> 修改方式：把最大堆和最小堆统一设为 4G，启动即锁定，完全消除运行期堆扩缩引发的额外 GC；GC 最大停顿时间设置为 MaxGCPauseMillis=200ms，G1 会据此自动调整每次回收的 Region 数量——每次少回收一点、多回收几次，而不是一次性大扫除，把 STW 停顿控制在 200ms 以内，避免单次 GC 把服务停顿数秒影响上层接口响应。

> 在 JVM 参数调优阶段，我顺带做了一个 GC 回收器的对比实验，想看看换 Parallel GC 能不能进一步提升性能。
> Parallel GC 是以吞吐量为优先的回收器，多线程并行执行 GC，CPU 利用效率更高；G1 是以低延迟为优先同时兼顾吞吐量，通过分 Region 增量回收，把每次停顿时间控制在目标范围内。
> 实验结果上 Parallel GC 吞吐量高约 4%，但平均停顿从 120ms 涨到 350ms，最大停顿从 350ms 涨到 850ms，差了将近 2.5 倍。
> 我们最终还是选了 G1，原因有两点：第一，系统核心是通过 K8s Informer 的 Watch 连接实时监听资源变更事件，长达 850ms 的 STW 停顿期间，JVM 完全暂停，Watch 连接无法处理任何事件，积压的事件队列会瞬间膨胀，严重时还可能导致 Watch 连接超时重连，重连后触发新一轮全量 List，反而进一步加大内存压力。第二，系统也对外提供了供 lb-sphere 调用的鉴权数据查询接口，850ms 的停顿会直接影响这些接口的响应时间。
> 所以，对整个系统来说，保持吞吐量与停顿时间的平衡是最重要的，4% 的吞吐量收益抵不过引入的风险，最终保留 G1。

#### 问题解析

**（1）LRU 淘汰会不会影响同步正确性？**

不会，因为这个 Map 的用途只是去重（防止同一资源被重复处理），不是业务数据。它存的是 resourceKey → 已处理标记，逻辑是：
~~~
if (processedMap.containsKey(key)) → 跳过
else → 处理，并写入 processedMap
~~~

即使某条旧的 Key 被 LRU 淘汰，最坏结果是同一资源被重新处理一次（重复调一次 saveOrUpdate），但因为底层数据库是幂等的 saveOrUpdate（存在则更新，不存在则插入），重复处理不会产生错误数据。数据正确性完全不受影响，只是极少情况下有一次冗余写入，代价完全可以接受。

**（2）CallerRunsPolicy 原理**

线程池满了之后，任务提交会被拒绝。CallerRunsPolicy 的做法是：不扔任务，让提交任务的那个线程自己去执行这个任务。
~~~
正常情况：  业务线程 → 提交任务 → 线程池执行
队列满时：  业务线程 → 提交任务 → 线程池拒绝 → 业务线程自己执行这个任务
~~~

这样做的效果是：
- 任务永远不丢（面试官最关心这点）
- 天然形成背压：业务线程被"征用"去干活，没法继续提交新任务，相当于把提交速度自动降下来，等线程池消化完积压再恢复

**（3）为什么出问题直接扩容到 8C16G 后面还是会出现问题？**

有人可能会问，为什么不直接扩容到 8C16G 解决问题？扩容确实能暂时缓解，因为堆更大，Full GC 触发的频率会变低，但根本问题没有解决。

无界 HashMap 仍然会无限增长，只是从填满 4G 变成填满 8G，时间问题；线程池队列仍然会积压，只是能多撑一段时间。而且随着集群持续增加、监听资源继续增长，迟早还会触到新的天花板。扩容是在给'内存漏桶'换个更大的桶，而不是堵住漏洞。

正确的顺序是：先扩容解决问题 → 治根因（有界集合、批处理）→ 再做 JVM 参数调优（让 GC 跑得更好）→ 最后适当扩容（为未来业务增长留余量）。我们最终也确实做了扩容，但那是在代码和参数都优化完之后，作为提升稳定性上限的最后一步，而不是作为救火的第一步。


### 经验总结（15 秒）

> "这次调优让我认识到，JVM 调优不止是调参数，根本是要先弄清楚'内存去哪了'。内存问题优先从代码逻辑（无界集合、对象生命周期）找根因，然后再通过 JVM 参数为回收策略提供更好的运行条件，两者缺一不可。"

---

## 附：关键 JVM 指标如何在容器环境里查看

| 指标 | 查看路径 |
|------|---------|
| 容器线程总数 | 容器云监控 → PROC → 线程数 |
| Full GC / Young GC 次数 | 应用监控 → 基础监控 → GC 查看 |
| GC 暂停时间（STW 时长） | 应用监控 → 基础监控 → GC 查看 |
| 线程池队列长度 | 应用监控 → 基础监控 → ThreadPool → 详情 |
| Heap Dump / Thread Dump | FullGC 告警自动触发上传 OSS；或平台手动触发 |
