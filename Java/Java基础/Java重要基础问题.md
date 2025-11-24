### 1、Java 中 BIO、NIO、AIO

Java里面的 BIO、NIO、AIO的区别如下:

![alt text](/Java/Java基础/img/image.png)

#### （1）BIO （同步阻塞）

经典的“一个连接一个线程”模型，服务端使用 ServerSocket.accept()阻塞等待客户端连接。连接建立后，为每个客户端分配一个线程，该线程调用 inputStream.read() 时会阻塞直到数据到达或连接关闭。数据读取是同步的，线程必须等 IO 操作完成才能继续执行。

BIO 实现最简单，调试方便，但是线程开销大，每个连接占用一个线程，
线程频繁阻塞，CPU 利用率低。

BIO 适用于连接数少（<1000）、业务简单、延迟要求不高的场景，如传统 JDBC 连接、简单的文件读写等。

**详细原理**：

一个连接一个线程的模型，在 BIO 中，服务器的 accept() 和客户端的 read()/write() 都是阻塞操作。当一个客户端连接后，服务器为其分配一个专用线程。这个线程在等待数据时（e.g., in.read()）会进入阻塞状态，即线程暂停执行，但仍占用系统资源（栈内存、线程上下文）。

阻塞线程不占用 CPU（OS 会调度到其他线程），但它占用线程槽位。问题在于，大量线程阻塞时，系统线程池饱和，新连接无法处理，导致队列积压或拒绝服务。

高并发下，OS 频繁切换线程（从阻塞到就绪），消耗 CPU 时间，这是上下文切换开销。

IO 操作慢（网络/磁盘），线程大部分时间在“空等”，无法复用。连接不释放，线程不释放；但即使连接活跃，如果 IO 间隙长，线程仍闲置。

#### （2）NIO（同步非阻塞 IO / New IO）

NIO 引入三大核心组件：Channel（双向通道）、Buffer（缓冲区）、Selector（多路复用器）。NIO 的特点如下：

- 非阻塞：channel.configureBlocking(false) 后，read()/write() 立即返回（可能返回 0）。
- 多路复用：一个线程通过 Selector 同时监听多个 Channel 的 IO 事件（可读、可写、连接、接受）。
- 当事件就绪时，Selector 返回对应 Channel，线程再处理（仍是同步：线程自己读写数据）

NIO 的**单线程可处理上万连接**（事件驱动），实际生产多用 多线程 Reactor 或框架（如 Netty）。

NIO 适用于高并发、连接数多但**每次交互数据量小**的场景，适用于聊天服务器、游戏服务器、实时推送、HTTP 服务器。大厂主流框架基础：Netty、Dubbo、Kafka、Tomcat NIO 连接器都基于 NIO。

**NIO 详细原理**：

- 线程复用：单线程 + Selector 处理所有连接。只有事件就绪（如数据可读）时，才分配时间片处理，线程不阻塞在单个连接上。
  
- 无连接占用线程：连接注册到 Selector 后，线程继续监听其他事件。如果无数据，线程不挂起，而是继续 select()。

- 上下文切换少：少线程 = 少切换。事件驱动模型（Reactor），CPU 利用率高。
- 
- 量化：NIO 可处理 10万+ 连接，BIO 限 1000 左右。



#### （3）AIO（异步非阻塞 IO）

AIO 是真正异步，IO 操作由操作系统内核完成，完成后通过回调或Future（轮询）通知 Java 线程。核心接口：AsynchronousSocketChannel、AsynchronousServerSocketChannel。

AIO 线程完全不参与实际 IO，CPU 利用率最高。Linux 下基于 epoll + 线程池模拟异步（性能不如原生），Windows 下 IOCP 原生支持最好。但是存在回调地狱严重，代码可读性差（Netty 4+ 已放弃 AIO）。

AIO 极高并发且每次 IO 操作耗时长的场景（如大文件传输、数据库异步查询）。Windows 平台高性能网络服务。不推荐在 Linux 上大规模使用（性能不如 NIO + epoll）。

### 2、零拷贝

零拷贝（Zero-Copy）是一种 IO 优化技术，旨在减少用户空间（Java 进程）和内核空间（OS 内核）间的数据拷贝次数，从而**降低 CPU 开销、上下文切换和延迟**。在传统 IO 中，数据传输需多次拷贝；零拷贝通过硬件和内核机制最小化拷贝，甚至实现“零” CPU 拷贝。

#### （1）传统 IO 的拷贝问题
流程（以文件 → 网络为例）：

- 磁盘数据 → 内核缓冲（DMA 拷贝，无 CPU）。
- 内核缓冲 → 用户缓冲（CPU 拷贝）。
- 用户缓冲 → 内核 socket 缓冲（CPU 拷贝）。
- 内核 socket 缓冲 → 网卡（DMA 拷贝）。

问题：2 次 CPU 拷贝 + 4 次上下文切换（用户-内核模式切换）。CPU 开销大，延迟高（大文件传输瓶颈）。

#### （2）零拷贝的原理

核心：避免不必要的 CPU 拷贝，让数据直接从源到目标（DMA + 内核机制）。

常见技术：
- mmap（内存映射）：将文件映射到用户空间地址，共享内核缓冲。读写像访问内存，避免内核-用户拷贝。但仍需 1 次 CPU 拷贝（到 socket 缓冲）。
- sendfile：内核直接从文件缓冲 → socket 缓冲（零 CPU 拷贝）。Linux 2.1+ 支持。
- splice：类似 sendfile，但更通用（管道间零拷贝）。

**DMA 的作用**：硬件直接内存访问，拷贝无 CPU 干预。

**零拷贝效果：拷贝减到 2 次（DMA），上下文切换减到 2 次。性能提升 2-3x（CPU 利用率高）**。

#### （3）与其他 IO 的关系

BIO：无零拷贝，传统拷贝多。
NIO：支持零拷贝，是其性能关键。
AIO：异步 + 零拷贝（如异步 transferTo），极致优化。


### 3、IO 多路复用

#### （1）原理

IO 多路复用是操作系统中一种高效的 I/O 处理机制，它允许一个进程或线程同时监控多个文件描述符的 I/O 事件状态，并在事件就绪时进行处理。这种机制的核心在于“复用”，即复用一个线程来管理多个 I/O 操作，避免了为每个 I/O 操作分配一个独立线程的开销。

在操作系统中，I/O 操作（如读写文件、网络数据传输）通常是慢速的，因为它们涉及硬件（如磁盘、网络接口）和内核态/用户态切换。

传统阻塞式 I/O（Blocking I/O）中，一个线程在等待 I/O 完成时会被阻塞，导致线程和 CPU 资源浪费。如果使用多线程为每个连接分配线程（如 Java BIO 模型），高并发场景下会引发线程爆炸。

IO 多路复用解决了这个问题，它通过系统调用让内核监控多个文件描述符（fd）的状态，进程/线程阻塞在多路复用调用上，而不是单个 I/O 操作上。当任意 fd 就绪时，内核通知进程处理。这实现了“事件驱动”模型，提高了系统效率和并发能力。

常见应用包括：

- 网络服务器：监听多个客户端连接。
- 文件系统：同时监控多个文件变更。
- 实时系统：处理多个传感器输入。

IO 多路复用是 NIO（Non-blocking I/O）的底层支撑，在操作系统层面由内核 API 实现，如 Linux 的 select、poll 和 epoll。

#### （2）实现机制

操作系统提供了几种 IO 多路复用 API，主要在 Linux 上演进：从 select 到 poll，再到 epoll。

##### （1）select

原理：select 是最早的多路复用 API，用户进程通过 select() 系统调用**传入三个 fd 集合（读、写、异常位图）**，内核拷贝这些集合到内核空间，并轮询检查每个 fd 的状态。如果有 fd 就绪，内核修改位图并返回就绪 fd 数量。用户进程再轮询位图处理就绪 fd。

系统调用签名（C 语言）：int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

内核实现：位图（bitmap）存储 fd，最大 1024 位（受 FD_SETSIZE 宏限制）。

**特点**：
- 支持超时（timeout 参数）。
- 每次调用需重新传入 fd 集合（内核-用户拷贝开销）。
- 复杂度：O(n)，内核每次遍历所有 fd。

优点：简单、跨平台（Windows 支持 Winsock select）。
缺点：fd 限制 1024；高并发下拷贝和遍历开销大；位图操作不高效。

适用场景：连接数少（<1024）、简单网络程序。早期 Java NIO 可能 fallback 到 select，但现在弃用。

与 IO 的关系：select 使非阻塞 IO 可能，fd 设置非阻塞后，select 等待就绪，再读写。

##### （2）poll

poll 是 select 的改进版，用户传入一个 **pollfd 结构体数组（包含 fd、感兴趣事件 revents 和返回事件 events）**，内核检查每个 fd 的状态，并修改 revents 返回就绪事件。用户遍历数组处理。

系统调用签名：int poll(struct pollfd *fds, nfds_t nfds, int timeout);
内核实现：动态数组存储 fd，无固定大小限制。

特点：
- 支持无限 fd（链表/数组扩展）。
- 事件类型丰富（e.g., POLLIN 可读、POLLOUT 可写）。
- 支持水平触发（LT）：事件未处理，下次 poll 仍报告。
- 每次调用仍需传入整个数组（拷贝开销）。
- 复杂度：O(n)，内核遍历所有 fd。

**优缺点：**
优点：无 fd 限制；事件分离（events/revents），更灵活。
缺点：仍需 O(n) 遍历；高并发拷贝大。

适用场景：中等连接数（几千）的服务器。Java NIO 在某些系统 fallback 到 poll。

与 IO 的关系：poll 增强了 select，**支持更多事件类型**，适用于更复杂的非阻塞 IO 场景。

##### （3）epoll

原理：epoll 专为高并发设计，用户先创建** epoll fd（epoll_create）**，然后通过 epoll_ctl 添加/修改/删除 fd 和事件。内核使用红黑树存储 fd，链表存储就绪事件。epoll_wait 阻塞等待，内核直接返回就绪事件数组（epoll_event 结构体）。

系统调用：
int epoll_create(int size);（创建 epoll fd，size 是提示大小）。
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);（op: ADD/MOD/DEL）。
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);（返回就绪事件数）。

内核实现：**红黑树（O(log n) 查找 fd）、就绪链表（O(1) 返回事件）、mmap 共享内存（零拷贝）。**

**特点：**
- 支持水平触发（LT）和边缘触发（ET）：ET 只通知一次变化，更高效但需小心（一次性读完数据）。
- 事件持久：fd 添加后永久监控，无需每次传入。
- 复杂度：O(1)（返回就绪数，不遍历所有 fd）。

**优缺点**：
优点：高并发支持（10万+ fd）；低开销；零拷贝。
缺点：Linux 专属；ET 模式编程复杂（易丢失事件）。

**适用场景**：大型服务器（如 Nginx、Redis）。Java NIO Selector 在 Linux 上用 epoll 实现。

**与 IO 的关系**：epoll 是现代 NIO/AIO 的基石，支持事件驱动 IO，解决 C10K/C100K 问题。

**详细的 epoll 工作流程：**
- **初始化**：进程调用 epoll_create，内核分配 epoll 对象（红黑树 + 就绪链表）。
-**注册 fd** ：epoll_ctl(ADD)，内核将 fd 插入红黑树，并注册回调到 fd 的驱动（如 socket 驱动）。
- **等待事件**：epoll_wait 阻塞进程。内核检查就绪链表，如果空则睡眠进程。
- **事件发生**：硬件中断（e.g., 网卡数据到达），fd 驱动调用 epoll 回调：
  - 检查 fd 是否在红黑树（已注册）。
  - 如果事件匹配（e.g., EPOLLIN），将 fd 插入就绪链表。
  - 唤醒 epoll_wait 的进程。

- **处理事件**：epoll_wait 返回就绪事件数组（从链表拷贝）。进程遍历数组，处理 fd（e.g., read(fd)）。
  
- **修改/删除**：epoll_ctl(MOD/DEL) 更新红黑树和回调。

#### （3）IO 多路复用的工作流程

- 注册 fd：进程将 fd 和事件注册到内核（e.g., epoll_ctl）。
- 等待事件：进程调用多路复用函数阻塞（e.g., epoll_wait），内核监控 fd。
- 事件就绪：硬件中断（如数据到达）触发内核检查 fd，通知进程。
- 处理事件：进程获取就绪 fd，执行读写（非阻塞模式下立即返回）。

#### （4）与其他 IO 模型的关系

- BIO：无多路复用，阻塞式。
- NIO：依赖多路复用（Selector 用 epoll），非阻塞同步。
- AIO：异步 + 多路复用（内核回调），但底层仍用 epoll-like 机制。

#### （5）文件描述符

文件描述符（File Descriptor，简称 fd）是操作系统中一个核心概念，**用于标识和管理打开的文件或资源**。它本质上是一个非负整数（通常从 0 开始），由内核分配给进程，用于访问底层资源（如**文件、套接字、管道、设备等**）。在 Unix-like 系统（如 Linux、macOS）和 Windows（称为句柄）中，fd 是进程与内核交互的“句柄”或“索引”。

### 4、反射

#### （1）反射原理

**反射的本质是**：一个 .class 文件的核心是**常量池包含字面量和符号引用）和属性表**，所有类名、方法名、注解、泛型信息都存在常量池里。在加载类时会把 .class 文件中的常量池和属性表解析成 C++ 的 InstanceKlass 结构，java.lang.Class 对象只是它的 Java 镜像。反射 API 实际上是对 InstanceKlass 中方法、字段、注解等信息的封装和拷贝，配合 setAccessible + 动态代理/字节码操作实现运行时动态访问私有成员。

.class 文件和 Class对象（instanceKlass 结构）如下：
![alt text](/Java/Java基础/img/image-1.png)

如下语句在反射的时候具体发生了什么：
~~~ java
// Java 层调用
Method[] methods = User.class.getDeclaredMethods();
~~~

![alt text](/Java/Java基础/img/image-2.png)

#### （2）反射的使用场景

- 依赖注入：Spring @Autowired → 反射 Field.set() 注入
- AOP：@Transactional、@Cacheable → 动态代理 + Method.invoke
- Web 路由：Spring MVC @GetMapping → 启动时反射扫描注册路由表
- ORM 映射：MyBatis/Hibernate → 反射把 ResultSet 塞到 private 字段
- JSON 序列化：Jackson/Fastjson → 反射调用所有 getter/setter
- 插件化/通用工具：Dubbo SPI、BeanUtils、JUnit、Lombok

#### （3）动态代理

**Java 动态代理 = 反射的“实战最强形态”**

Java 动态代理 = 在程序运行期间，JVM 根据接口动态生成一个**实现类字节码**，然后加载到 JVM 中，这个生成的实现类就是代理对象。

官方只有一种实现方式：java.lang.reflect.Proxy + InvocationHandler

~~~ java
// 核心代码就这几行
UserService target = new UserServiceImpl();

UserService proxy = (UserService) Proxy.newProxyInstance(
    target.getClass().getClassLoader(),
    target.getClass().getInterfaces(),   // 必须是接口！
    new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("日志开始...");
            Object result = method.invoke(target, args);  // ← 反射调用！
            System.out.println("日志结束");
            return result;
        }
    });
~~~

运行后你会发现：**根本没有写代理类**，但确实多了一个能打印日志的代理对象！

- Proxy.newProxyInstance 内部用了反射拿到所有接口的 Method 数组
- 生成的代理类里每个方法都是：handler.invoke(this, Method对象, args)
- 真正执行目标方法时，必须用 method.invoke(target, args) → 这就是反射

**总结**：JDK 动态代理的核心是 Proxy.newProxyInstance，它会在运行时**根据接口动态生成一个实现类的字节码**（继承 Proxy，实现你的接口），然后加载到 JVM。生成的代理对象里所有方法最终都会走到你传入的 InvocationHandler 的 invoke 方法里。我们在 invoke 里通过 method.invoke(target, args) 使用反射调用真实对象的方法，并在调用前后加入增强逻辑。这就是 Spring AOP、MyBatis Mapper 接口、Dubbo 接口调用等框架实现切面的底层原理。

可以说：没有反射就没有动态代理，没有动态代理就没有 Spring AOP、MyBatis Mapper 接口、Dubbo 接口调用这些现代 Java 框架的核心特性。

