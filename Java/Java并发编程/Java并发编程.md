## 一、面试场景题

### 1、上下文切换

#### （1）什么时候回发生上下文切换

- 时间片用完
- 阻塞操作 (Blocking I/O)：
    - 线程执行了 Thread.sleep()、Object.wait()、Thread.join()。
    - 线程发起了网络请求或读写磁盘，数据还没回来，线程主动挂起等待。
- 锁竞争 (Lock Contention)：线程尝试获取 synchronized 锁失败，被阻塞挂起。
- 高优先级抢占：突然来了一个优先级更高的线程，当前线程被迫让路。

#### （2）上下文切换的代价（为什么它慢？）

虽然一次切换只需要几微秒到几十微秒，看起来很快，但在高并发下（比如每秒发生几万次切换），累积起来的开销非常可怕。

- **CPU 算力的浪费**：保存和恢复寄存器、栈信息需要 CPU 指令。在切换期间，CPU 像是在“空转”，没有执行任何业务代码。

- **缓存失效**：CPU 有 L1/L2/L3 高速缓存。当线程 A 运行时，缓存里热乎乎的全是 A 的数据。一旦切换到线程 B，B 的数据会把缓存里 A 的数据挤走。等 A 再次回来时，缓存全冷了，必须去慢速的内存（RAM）里重新读取数据。这会导致代码执行速度瞬间变慢。

#### （3）如何减少上下文切换

- **无锁并发，减少并发环境下锁的使用**：使用 CAS (如 AtomicInteger) 代替 synchronized。因为 CAS 失败通常是自旋（Loop），线程不需要挂起，也就不需要上下文切换。

- **使用最少线程数** ：不要为每个请求都 new Thread。线程越多，分时间片的人越多，切换越频繁。最佳实践：如果是计算密集型任务，线程数 = CPU 核心数 + 1。这能保证 CPU 几乎没有切换，一直全速运算。

- **使用协程 / 虚拟线程**

### 2、Java 里面如何保证线程安全

保证 Java 线程安全的核心思想只有一句话：“**控制对共享变量的访问**”。

#### （1）互斥同步

也就是我们常说的“加锁”。同一时刻只允许一个线程访问共享资源。

- synchronized：这是 Java 原生语法层面的锁，由 JVM 实现，简单、自动释放锁
- ReentrantLock：这是 JDK 层面的锁，比 synchronized 更灵活（**支持尝试获取锁、公平锁、中断等待**）。

#### （2）非阻塞同步-CAS

利用 CPU 硬件指令（CAS - Compare And Swap）来保证原子性，不需要挂起线程，效率比加锁高。
如原子类 (AtomicInteger, AtomicLong 等)的实现就是利用 CAS 乐观锁机制。如果更新失败，就自旋重试。

#### （3）使用线程安全的集合 (Concurrent Collections)

- ConcurrentHashMap：分段锁 / CAS + Synchronized 节点锁，高并发读写。
- CopyOnWriteArrayList：读写分离，适合读多写少的场景。

#### （4）线程封闭

既然多线程抢资源不安全，那我就把资源私有化，每个线程一份，互不干扰，自然就安全了。如 ThreadLocal，常用于**数据库连接管理、SimpleDateFormat（因为它是线程不安全的）、用户 Session 信息**。

~~~ java
public class DateUtil {
    // 为每个线程分配一个独立的 SimpleDateFormat 对象
    private static final ThreadLocal<SimpleDateFormat> dateFormatHolder = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public static String format(Date date) {
        // 获取当前线程独享的那个 formatter
        return dateFormatHolder.get().format(date);
    }
}
~~~

#### （5）不可变对象
如果一个对象创建之后状态就不能变（Immutable），那它天生就是线程安全的，随便多少个线程同时读都没事。如 final 关键字与不可变类。

总结：“保证线程安全主要有三个维度的手段：

**控制并发访问**：最常用的手段。如果是复杂的业务逻辑，我会用 synchronized 或 ReentrantLock 加锁；如果是简单的计数或状态更新，我会用 Atomic 原子类（CAS）来提升性能。

**使用并发工具**：数据结构上，我会直接使用 ConcurrentHashMap 或 CopyOnWriteArrayList，避免自己造轮子。

**避开共享状态**：从设计上规避竞争。比如使用 ThreadLocal 做线程隔离，或者设计 不可变对象 (Immutable Objects)，这样根本就不需要同步。”

### 3、JDK1.6 synchronized做了哪些优化

- **锁升级机制**：无锁-> 偏向锁 -> 轻量级锁 -> 重量级锁
- **自适应自旋**：在 JDK 1.6 之前，也有自旋，但次数是固定的（比如 10 次）。如果转了 10 次没抢到，就挂起。JDK 1.6引入预测机制：
  - 如果这个锁之前刚刚被成功自旋拿到过，JVM 会认为“这次也有戏”，于是允许你多转几圈（增加自旋次数）。

  - 如果这个锁很少被自旋拿到（持有者通常执行很久），JVM 就会认为“别浪费 CPU 了”，直接挂起，甚至取消自旋。
- **锁消除**：JIT 编译器在编译代码时，通过逃逸分析 (Escape Analysis) 技术，判断同步块使用的锁对象是否只能被一个线程访问（没有逃逸出当前方法）。如果判断该锁不可能被共享，JIT 就会直接把这里的 synchronized 抹掉，完全不执行加锁逻辑。
- **锁粗化**：如果 JVM 检测到有一连串的操作都对同一个对象加锁（例如在循环体内加锁），或者频繁地进行加锁解锁操作。

### 4、死锁产生的条件已经如何避免

死锁就是：两个或多个线程互相持有对方想要的资源，同时又在等待对方释放资源，导致所有人都无法继续执行。只有同时满足以下 4 个条件，死锁才会发生。只要打破其中任意一个，死锁就不会发生。

- **互斥条件**：资源是独占的，同一时刻只能被一个线程持有（比如 synchronized 锁）
- **请求与保持条件**：一个线程因请求资源而阻塞时，对已获得的资源保持不放
- **不可剥夺条件**：线程已获得的资源，在未使用完之前，不能被其他线程强行剥夺，只能由自己释放。
- **循环等待条件**：若干线程之间形成一种头尾相接的循环等待资源关系

如何避免死锁：

- **破坏“循环等待”** —— 统一锁顺序，规定所有线程获取锁的顺序必须一致 (最推荐)
- **破坏“不可剥夺”** —— 使用 tryLock (ReentrantLock)，synchronized 是死等的，一旦拿不到锁就一直挂起。 而 ReentrantLock 提供了 tryLock() 方法。策略：如果尝试获取锁失败（或超时），我主动释放我已经持有的锁，过一会再重试。
- **破坏“请求与保持”** —— 一次性申请所有资源
- **减小锁粒度 & 避免嵌套锁**：尽量不要在持有一个锁的同时去申请另一个锁。锁住的代码块越小越好。

### 5、线程池常见类和接口之间的关系

线程池里面常见的类和接口包括：
- Executor (顶层接口) 
- ExecutorService (核心接口) 
- AbstractExecutorService (抽象基类) 
- ThreadPoolExecutor (核心实现类) 
- ForkJoinPool
- Executors (工厂类，负责创建它) 
- ScheduledThreadPoolExecutor

他们之间的关系如下图：
![alt text](/Java/Java并发编程/img/image.png)

FixedThreadPool、SingleThreadExecutor、CachedThreadPool 并不是类名（在标准 API 中），它们是 Executors 工厂类生产出来的特定配置的 ThreadPoolExecutor 对象，返回使用一个ExecutorService对象承接，生产环境通常禁止使用它，建议直接 new ThreadPoolExecutor。

示例源码如下：

![alt text](/Java/Java并发编程/img/image-1.png)

### 6、JUC 总结图谱

#### （1）地基与核心引擎

**CAS (Compare And Swap)**
- 实现：sun.misc.Unsafe 类。
- 作用：无锁原子操作的基础。
- 问题：ABA 问题（解法：版本号 AtomicStampedReference）

**Volatile**
- 作用：保证可见性（MESI 缓存一致性协议）、有序性（禁止指令重排内存屏障）。
- 局限：不保证原子性。

**AQS (AbstractQueuedSynchronizer)**
- 本质：一个 int state (资源) + 一个 CLH 双向链表 (排队) + LockSupport (挂起/唤醒)。
- 模式：独占模式 (Exclusive) + 共享模式 (Shared)。
- 模板方法：子类实现 tryAcquire/tryRelease

#### （2）锁与同步器

基于 AQS 构建的访问控制机制。

**ReentrantLock (独占锁)**
- 特点：可重入、可中断、支持超时、支持公平/非公平。
- 核心：AQS 独占模式。
- Condition：替代 wait/notify，实现分组唤醒。

**ReentrantReadWriteLock (读写锁)**
- 特点：读读共享，读写互斥。
- 核心：State 高16位存读锁，低16位存写锁。
- 问题：写锁饥饿，不支持锁升级，支持锁降级。

**LockSupport**
- 作用：线程阻塞原语 (park/unpark)，AQS 的底层工具。

#### （3）原子类 (Atomics)

- 基本类型：AtomicInteger, AtomicLong, AtomicBoolean。
- 数组类型：AtomicIntegerArray 等。
- 引用类型：AtomicReference (对象原子更新)、AtomicStampedReference (带版本号，防 ABA)、AtomicMarkableReference (带标记)。
- 字段更新器：AtomicIntegerFieldUpdater (省内存，原子修改 volatile 字段)。
- 高性能累加器 (JDK 8)：LongAdder / DoubleAdder。原理：分段锁思想（Cell 数组），空间换时间，解决高并发自旋瓶颈。

#### （4）并发工具类

基于 AQS 共享模式实现的线程协作工具。

**CountDownLatch (减法)**
口诀：秦灭六国，一统天下（主线程等 N 个子线程）。不可重用，基于AQS。

**CyclicBarrier (加法)**
口诀：集齐七颗龙珠召唤神龙（N 个线程互相等，人齐了继续）。可重用，基于 ReentrantLock。

**Semaphore (信号量)**
口诀：抢车位（限流，控制并发数量），基于AQS，state 代表许可证数量。

#### （5）并发容器 (Collections)

线程安全的数据结构。

- Map
  - ConcurrentHashMap 🔥：1.7：Segment 分段锁。1.8：Node + CAS + Synchronized。
   - ConcurrentSkipListMap：跳表实现，有序的 Map (类似 TreeMap 的并发版)。

- List / Set
    - CopyOnWriteArrayList：写时复制，读写分离。适合读多写少。
    - CopyOnWriteArraySet：底层基于 CopyOnWriteArrayList。

- Queue (非阻塞)
    - ConcurrentLinkedQueue：基于链表，全程 CAS 无锁。

#### （6）阻塞队列 (BlockingQueue)

生产者-消费者模型，线程池的血管。

- ArrayBlockingQueue：数组，有界，一把锁。
- LinkedBlockingQueue：链表，有界（默认无界坑），两把锁（读写分离）。
- SynchronousQueue：容量为0，手递手，不存储。
- PriorityBlockingQueue：支持优先级的无界阻塞队列。

#### （7）线程池与执行框架

接口：Executor -> ExecutorService。

实现类：
- ThreadPoolExecutor 🔥：全能选手，7 大参数。
- ScheduledThreadPoolExecutor：定时任务。
- ForkJoinPool：分治算法，工作窃取 (Work Stealing)。

辅助类：Executors (不推荐直接使用，易 OOM)。

任务与结果：
- Runnable (无返回值)。
- Callable (有返回值)。
- Future (异步结果)。
- CompletableFuture (JDK 8)：异步编排，链式调用，回调地狱克星。

#### （8）记忆
- 底层靠 CAS 和 Volatile 撑起了一片天。
- AQS 拿这俩做地基，造出了 ReentrantLock 和各种 工具类(Latch/Semaphore)。
- 为了不加锁也能安全，搞出了 Atomic 和 LongAdder。
- 为了存数据安全，把 HashMap 改造成了 ConcurrentHashMap，把 List 改成了 CopyOnWrite。
- 为了把任务和线程解耦，搞出了 BlockingQueue，并以此为基础搭建了 ThreadPoolExecutor。
- 最后为了玩转异步，弄出了 Future 和 CompletableFuture。






