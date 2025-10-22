## 一、常见主要问题
### 1、JVM 里面常见的 3 种常量池
Class 文件常量池、运行时常量池、字符串常量池。

关注运行时常量池和字符串常量池随着 JDK 版本更新所在位置的变化。

**相互关系**：Class 文件常量池 → 加载后形成运行时常量池 → 字符串常量池从中分离（JDK 7+）。它们共同支持 JVM 的“类加载-链接-初始化”过程。

各种方式创建的字符串在堆中的存储位置。

### 2、元空间实现的方法区的回收机制

方法区存储的类信息（类元数据）、运行时常量池、静态变量、JIT 即时编译器编译的内容。

JDK8之前永久代实现的方法区内容的回收，是在 Full GC 的时候进行回收，每次 Full GC 都会对方法区的内容进行回收，效率低，GC 的 STW 时间较长。

元空间实现的方法区，Full GC 期间进行类卸载检查和回收（类似于永久代），但不是每次 GC 都触发。JVM 会延迟检查以减少开销，只有当元空间使用率高（接近阈值）时，才在 Full GC 中执行卸载。
虽然类卸载通常在 Full GC 中发生，但元空间 GC 不像像永久代那样“强制 Full GC”，而且回收时不扫描整个堆，只针对类加载器和引用链，速度比永久代快 2-5 倍。

![alt text](/Java/JVM/img/image.png)

### 3、类卸载的条件

- **无类实例存在**: 该类的所有对象实例（包括数组）已被 GC 回收，无任何强引用指向它们。
- **无 Class 对象引用**: 该类的 Class 对象（通过 Class.forName() 或反射获取）无任何引用。
- **类加载器无引用链**: 加载该类的 ClassLoader（类加载器）本身无引用，且该加载器加载的所有其他类无引用。
  
### 4、类创建的过程

JVM 里面常见一个类（new）包括：**类加载检查、内存分配、对象头初始化、实例变量赋值 和 构造函数执行。** 这个过程从 new 指令或反射调用开始，确保对象在堆中正确分配内存并初始化数据。

#### （1）类加载检查

- 执行 new 指令时，先从常量池获取符号引用；
- 通过类加载器（ClassLoader）检查类是否已加载（加载 → 链接 → 初始化），如果未加载，触发类加载：Bootstrap/Extension/Application Loader 加载 .class 文件，生成 Class 对象。


#### （2）内存分配

在堆（Heap）中分配对象大小（对象头 + 实例字段 + 对齐填充）。

分配方式：
  - 指针碰撞（Bump the Pointer）：空闲内存连续时，移动指针。
  - 空闲列表（Free List）：内存碎片时，维护链表分配。
  
并发安全：使用 CAS（Compare-And-Swap）或 TLAB（Thread Local Allocation Buffer，线程本地缓存）预分配。

#### （3）对象头初始化 

设置对象头（Object Header）：
  - Mark Word（64 位）：哈希码、GC 分代年龄、锁标志、偏向锁等（动态格式）。
  - Klass Pointer：指向方法区 Class 对象（类型信息）。

#### （4）实例变量赋值

为所有实例字段（非静态）赋默认值（int=0, boolean=false, 引用=null 等）

#### （5）构造函数执行

执行 <init>() 方法（实例初始化方法），包括：调用父类 <init>（super()）、执行实例代码块、赋值显式字段值、调用用户构造函数。完成后，返回对象引用给调用者。

### 5、指针碰撞和空闲列表这两种内存分配算法如何保证分配内存时候的线程安全

在 JVM（HotSpot 实现）中，内存分配是多线程环境下的高频操作，因此指针碰撞和空闲列表算法都需确保线程安全，避免并发分配导致的内存覆盖或数据不一致。

主要通过 **原子操作（如 CAS）和线程本地缓冲（TLAB）** 机制实现。TLAB 是核心优化：每个线程预分配一块私有缓冲区，在缓冲内分配无需同步；缓冲耗尽时，再从全局堆原子申请新 TLAB。

#### （1）指针碰撞

**TLAB 机制：**

每个线程在 Eden 区（新生代）预申请一个 Thread Local Allocation Buffer（TLAB），大小通常为 16KB-1MB。在 TLAB 内，使用指针碰撞分配：线程独占缓冲，无需锁，直接原子更新本地指针（bump-the-pointer）。
TLAB 耗尽时，线程使用 CAS（Compare-And-Swap）从全局 Eden 区原子申请新 TLAB。如果申请失败，退化为全局同步。

优点：分配路径快（几条指令），减少锁竞争；GC 时，TLAB 未用空间直接丢弃。

**CAS 原子操作（全局回退）**：

当无 TLAB 或大对象分配时，使用 CAS 比较并交换全局空闲指针，确保只有一个线程成功移动指针，避免覆盖。

#### （2）空闲列表

**细粒度锁或 synchronized**：

JVM 对空闲列表加 synchronized 锁（Java 层）或 OS 互斥锁（底层），确保原子访问链表。优点是实现简单，缺点是锁粒度大时竞争激烈。
**分配过程**：加锁 → 搜索合适块（First Fit/Best Fit） → 取出块 → 解锁。
**回收时**：加锁 → 插入碎片块 → 解锁。


**CAS 原子操作（优化路径）**：

适用于低竞争场景，重试机制处理失败。

### 6、对象主要包含哪几部分

普通对象主要由对象头、实例数据和对齐填充组成，数组对象额外包含数组长度字段。对象大小通常为 8 字节对齐（64 位 JVM），总大小 = 对象头 + 实例数据 + 填充。

![alt text](/Java/JVM/img/image1.png)

### 7、内存泄露的原因

#### （1）静态集合不当使用
静态集合持有大量无用对象引用，未及时清理，导致 GC 无法回收（静态对象只有在类卸载的时候才会被回收，类卸载的要求较为严格）。

**案例描述**：静态 HashMap 不断添加用户对象，但未移除过期项，导致所有对象无法被 GC 回收。
~~~ java
import java.util.HashMap;
import java.util.Map;

public class StaticCollectionLeak {
    private static final Map<String, User> userCache = new HashMap<>();  // 静态集合

    static class User {
        String name;
        User(String name) { this.name = name; }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 100000; i++) {  // 模拟大量添加
            userCache.put("user" + i, new User("User" + i));  // 未移除，泄漏
            System.out.println("Added user" + i);
            if (i % 10000 == 0) {
                try { Thread.sleep(100); } catch (InterruptedException e) {}  // 延时观察
            }
        }
    }
}
~~~

#### （2）未关闭的资源（字节流、Socket、数据库连接）

资源对象持有外部引用（如文件句柄、连接池），未调用 close()，导致引用链断不开。

**案例描述**：在循环中打开文件输入流，但未关闭，导致流对象和底层文件句柄持续占用内存。

~~~ java
import java.io.FileInputStream;
import java.io.IOException;

public class ResourceLeak {
    public static void main(String[] args) {
        for (int i = 0; i < 10000; i++) {  // 模拟多次打开文件
            try {
                FileInputStream fis = new FileInputStream("test.txt");  // 未 close()
                byte[] buffer = new byte[1024];
                fis.read(buffer);  // 读取但不关闭
                System.out.println("Read file " + i);
            } catch (IOException e) {
                e.printStackTrace();
            }
            // 缺少 fis.close(); 导致泄漏
        }
    }
}
~~~

#### (3) 长生命周期对象持有短生命周期引用

案例描述：静态单例持有临时请求对象列表，未释放，导致所有请求对象存活。

#### (4) 线程池或线程未正确关闭

案例描述：创建 ExecutorService，但未调用 shutdown()，导致线程和任务对象积累。
~~~ java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolLeak {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(10);  // 未 shutdown()

        for (int i = 0; i < 100000; i++) {
            final int taskId = i;
            executor.submit(() -> {  // 提交任务，但线程池不关闭
                try { Thread.sleep(10); } catch (InterruptedException e) {}
                System.out.println("Task " + taskId + " executed");
            });
        }
        // 缺少 executor.shutdown(); 导致线程和任务引用泄漏
    }
}
~~~













## 二、常见案例以及回答

### 1、开发的过程种有没有遇到过内存泄露，是怎么检测的，怎么解决的































































