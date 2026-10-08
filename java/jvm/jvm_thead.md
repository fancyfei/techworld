# JVM 线程架构

> Java 线程与内核线程是 1:1 映射：`java.lang.Thread` 是堆上的普通对象，真正驱动执行的是 HotSpot 内部的 C++ 对象 `JavaThread` 与其绑定的操作系统线程。

## 一、线程模型

### 1.1 三层结构

JVM 本身不具备 CPU 调度能力，Java 线程采用 **1:1 线程模型**。

```
java.lang.Thread        (堆上的 Java 对象，只是句柄)
        │  JNI: start0()
        ▼
JavaThread  (HotSpot C++ 对象，JVM 管理线程的实体)
        │  持有 OSThread
        ▼
OSThread  (封装 pthread_t / HANDLE，记录 OS 调度状态、栈信息)
        │
        ▼
内核线程 (pthread_create 产物，OS 调度的实体)
```

- **Java Thread (Java 层)**：`java.lang.Thread` 对象，是 JVM 暴露给开发者的 API 封装。
- **JVM Thread (C++ 层)**：JVM 内部的 `JavaThread` 对象，继承自 `Thread` 类，维护 JVM 级别的线程状态、栈帧（Stack Frame）指针、JNI 局部变量表以及安全点（Safepoint）标志。
- **OS Thread (内核层)**：Linux 下的 NPTL（Native POSIX Thread Library）线程，由系统的 `pthread_create()` 创建，本质上是内核调用 `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND)` 创建的轻量级进程（LWP）。

### 1.2 虚拟线程（Virtual Thread）

Java 21 引入了虚拟线程（Virtual Thread），改写了原有的 1:1 映射机制，实现了 **M:N 协程模型**。

- **虚拟线程（Virtual Thread）**：由 JVM 在用户态管理的轻量级线程。其实例为 `java.lang.VirtualThread`（继承自 `Thread`），不直接绑定内核线程。
- **载体线程（Carrier Thread）**：虚拟线程运行在平台线程之上，平台线程作为 Carrier Thread。当虚拟线程执行传统非阻塞/可剥离的阻塞操作（如 `java.nio` 或 `LockSupport.park()`）时，JVM 会剥离（Unmount）其上下文，将 Carrier Thread 释放给其他虚拟线程使用；当阻塞结束，虚拟线程被重新调度（Mount）到任意可用的 Carrier Thread 上。

## 二、Java 线程状态与 OS 状态的映射

`java.lang.Thread.State` 定义了 Java 语言维度的 6 种线程状态，它与操作系统底层的内核线程状态并不一一对应。

### 2.1 Thread.State 六态

```java
public enum State {
    NEW,           // new Thread() 后未 start()
    RUNNABLE,      // 可运行（含 OS 运行态/就绪态，甚至 IO 阻塞）
    BLOCKED,       // 等待进入 synchronized 监视器
    WAITING,       // wait()/join()/park()，无限期
    TIMED_WAITING, // sleep(ms)/wait(ms)/parkNanos()，带超时
    TERMINATED     // run() 结束
}
```

### 2.2 映射：粗粒度抽象的设计

Java 状态是 JVM 对 OS 调度状态的粗粒度封装，并非一一对应：

| Java 状态                         | Linux 状态      | 对应关系                         |
| --------------------------------- | --------------- | -------------------------------- |
| NEW                               | 无              | 尚无内核线程                     |
| RUNNABLE                          | R（运行/就绪）  | 合并了运行态与就绪态             |
| BLOCKED / WAITING / TIMED_WAITING | S（可中断睡眠） | 挂起等待唤醒，JVM 细分了等待原因 |
| RUNNABLE（IO 阻塞中）             | S 或 D          | JVM 对 native 阻塞无感知         |
| TERMINATED                        | Z → 回收        | 生命周期终结                     |

映射表体现的设计取舍值得注意：JVM 只关心"从 Java 程序的视角，这个线程在逻辑上处于什么阶段"，不关心 OS 层的调度细节。

## 三、 线程运行、调度与上下文切换

### 3.1 CPU 时间片调度与线程切换

HotSpot 在 Linux 上采用抢占式调度模型，完全依赖操作系统的调度器（如 Linux CFS）。

- JVM 不干预线程的时间片分配，`Thread.yield()` 仅是对 OS 调度器的一个 hint（在 Linux 下通过 `sched_yield()` 系统调用实现），调度器可以忽略该请求。
- `Thread.setPriority()` 映射为操作系统的 `nice` 值。在 Linux 上，普通用户权限下调整优先级往往效果受限或被忽略，无法保证高优先级的线程一定先执行。

### 3.2 Thread Stack 结构与 Thread Context

每个 Java 线程在创建时都会由操作系统分配一块独立的线程栈空间（通过 `-Xss` 参数控制，默认一般为 1024KB）。线程栈由多个栈帧（Stack Frame）组成，对应每次调用的 Java 方法。

```
+-----------------------------------+  高地址
|           Method B Frame          |
|  - Local Variable Table (局部变量表)|
|  - Operand Stack (操作数栈)       |
|  - Frame Data (动态链接、返回地址) |
+-----------------------------------+
|           Method A Frame          |
+-----------------------------------+  低地址 (栈向下增长)
```

**线程运行上下文（Thread Context）** 包含：

- **CPU 寄存器状态**：程序计数器（PC，指向当前执行的字节码/机器码指令）、栈指针（SP）、帧指针（FP）、通用寄存器（AX, BX...）。
- **栈内存**：当前线程栈空间内的所有栈帧数据。
- **TLS (Thread Local Storage)**：指向 `ThreadLocal` 映射表及 JVM `JavaThread` 结构的指针（Linux 下通过 `FS`/`GS` 段寄存器实现快速寻址）。

### 3.3 上下文切换的开销

操作系统将 CPU 从一个线程切换到另一个线程，需要经历从用户态到内核态的变更，开销主要体现在：

1. **直接开销**：
   - 保存当前线程的 CPU 寄存器上下文至内核栈/PCB。
   - 运行 OS 调度算法选择下一线程。
   - 恢复下一线程的 CPU 寄存器上下文并切换页表指针（若涉及进程切换）。
2. **间接开销**：
   - **CPU Cache 失效**：新线程的数据不在 L1/L2/L3 Cache 中，产生大量 Cache Miss。
   - **TLB 命中率下降**：如果新线程访问的不同虚拟地址区间会导致 TLB 重新载入。
   - **流水线冲刷**：CPU 分支预测失效，引发指令流水线重置。

## 四、JVM 内部线程管理

JVM 启动（`Threads::create_vm`）后会创建一批非用户线程，`jstack` 可见：

| 线程                                           | 职责                             |
| ---------------------------------------------- | -------------------------------- |
| main                                           | 用户入口                         |
| Reference Handler / Finalizer / Common-Cleaner | 引用与资源清理                   |
| Signal Dispatcher                              | 信号处理                         |
| VM Thread                                      | 执行 VM 级操作（GC、线程转储等） |
| GC Thread / G1 并发线程                        | 垃圾收集                         |
| C1/C2 CompilerThread                           | JIT 编译                         |

*当 `VM Thread` 准备执行需要 Stop-The-World (STW) 的操作（如 GC）时，会设置 Safepoint 标志，所有 `JavaThread` 在轮询点（Polling Page）检测到标志后主动悬挂自己。*