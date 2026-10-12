# Java 线程池

现在的 CPU 都是多核的，用多线程并行计算提升服务器性能是基本手段。但操作系统创建和销毁线程需要时间，于是出现了基于池化思想管理线程的工具——线程池。

- 线程池的目的：执行提交给线程池的任务，执行完一个任务后不退出，而是继续等待或执行新任务。
- 线程池的组成：一个任务队列和一组工作者线程。任务队列保存待执行的任务，工作者线程循环从队列中取出任务并执行。

## 线程池的好处

- 降低资源消耗：重用线程，避免频繁创建和销毁线程的开销；
- 提高响应速度：任务复用已有线程，省去创建线程的过程；
- 管控并发数：可以限制线程池中的线程数量；
- 削峰填谷：任务排队处理，缓冲负载高峰的冲击，一定程度上防止系统过载；
- 提供延时、定时等更强的功能。

## Java 的 Executor

Java 并发包提供了线程池管理类 Executor，其核心思想是把任务提交和任务执行解耦：把 Runnable 任务提交给 Executor 框架，由它完成线程的调配和执行。

线程池内部基于生产者-消费者模式实现，通过一个阻塞队列衔接两者：提交线程把任务放进阻塞队列缓存，工作线程从队列中取出任务执行。

Executor 的实现类是 ThreadPoolExecutor。线程池内部用一个变量维护两个值：运行状态（runState）和线程数量（workerCount）。该变量是 AtomicInteger 类型，用高位和低位分别表示两个值。

核心构造器与变量是：

```java
public ThreadPoolExecutor
    (int corePoolSize,
     int maximumPoolSize,
     long keepAliveTime,
     TimeUnit unit,
     BlockingQueue<Runnable> workQueue,
     ThreadFactory threadFactory,
     RejectedExecutionHandler handler);
//任务缓存队列，用来存放等待执行的任务
private final BlockingQueue<Runnable> workQueue;
//线程池的主要状态锁
private final ReentrantLock mainLock = new ReentrantLock();
//用来存放工作集合
private final HashSet<Worker> workers = new HashSet<Worker>();
//线程存活时间
private volatile long  keepAliveTime;
//核心线程数
private volatile int   corePoolSize;
//最大线程数
private volatile int   maximumPoolSize;
//线程池中当前的线程数
private volatile int   poolSize;
//任务拒绝策略
private volatile RejectedExecutionHandler handler;
//线程工厂，用来创建线程
private volatile ThreadFactory threadFactory;
```

## 线程池的生命周期

运行状态展现了线程池的生命周期：

![thread_pool_lifecycle](thread_pool_lifecycle.png)

- RUNNING：能够接收新任务，并处理已添加的任务。
- SHUTDOWN：不接收新任务，但能处理已添加的任务。
- STOP：不接收新任务，不处理已添加的任务，并中断正在处理的任务。
- TIDYING：所有任务已终止，进入此状态后会调用钩子函数 terminated()。
- TERMINATED：线程池彻底终止。

## 线程池的重要参数

- corePoolSize（核心线程数）：正常保持的线程数，超出此数的任务先进入缓存队列。
- maximumPoolSize（最大线程数）：核心线程和队列都满后，还能继续创建到的线程数上限。
- keepAliveTime（线程存活时间）：超出核心线程数的那部分线程空闲时的最大存活时间。
- workQueue（任务队列）：用于传输和保存等待执行任务的阻塞队列。
- threadFactory（线程工厂）：用于创建新线程。
- handler（拒绝策略）：线程池和队列都满时，新任务的处理方式。

## 阻塞队列种类

参数 workQueue 的类型是阻塞队列 BlockingQueue，常见的可以用作线程池队列的有：

- ArrayBlockingQueue：基于数组的有界阻塞队列。
- LinkedBlockingQueue：基于链表的阻塞队列，可以指定最大长度，默认无界。
- SynchronousQueue：没有实际存储空间的同步阻塞队列，插入操作必须等待对应的移除操作。
- PriorityBlockingQueue：支持优先级的无界阻塞队列，任务按优先级出队而不是按提交顺序。
- DelayedWorkQueue：基于优先堆的无界阻塞队列，任务按延时时间出队，ScheduledThreadPool 使用。

线程池的任务排队和拒绝策略与 BlockingQueue 的选择是关联的。

## 拒绝策略

拒绝策略可以自定义，ThreadPoolExecutor 内置了四种实现：

- ThreadPoolExecutor.AbortPolicy：默认方式，直接抛出异常。
- ThreadPoolExecutor.DiscardPolicy：静默忽略新任务，不抛异常，也不执行。
- ThreadPoolExecutor.DiscardOldestPolicy：丢弃等待时间最长的任务，然后尝试把新任务加入队列。
- ThreadPoolExecutor.CallerRunsPolicy：由提交任务的线程自己执行该任务，而不是交给线程池中的线程。

## Executors 实现的线程池

Executors 是一个工厂类，可以创建常见场景的线程池，共六种：

- FixedThreadPool(n)：线程数固定为 n 的线程池，超出的任务在容量为 Integer.MAX_VALUE 的无界 LinkedBlockingQueue 中排队等待，可用于控制程序的最大并发数。
- CachedThreadPool()：适合短时间内处理大量任务的线程池，核心线程数为 0，最大线程数为 Integer.MAX_VALUE，任务在 SynchronousQueue 中直接交给空闲线程，空闲线程存活 60 秒后回收。
- SingleThreadExecutor()：只有一个工作线程的线程池，其余任务在 LinkedBlockingQueue 中排队，保证任务按提交顺序执行。
- ScheduledThreadPool(n)：核心线程数固定为 n、最大线程数为 Integer.MAX_VALUE 的线程池，支持定时和周期性任务，队列是 DelayedWorkQueue。
- SingleThreadScheduledExecutor()：单线程版的 newScheduledThreadPool。
- WorkStealingPool(n)：Java 8 新增的方法，基于 ForkJoinPool 创建抢占式并行线程池，空闲线程可以从其他线程的队列中窃取任务，不保证任务的执行顺序。

工程上的普遍做法是直接构造 ThreadPoolExecutor 并显式指定有界队列和拒绝策略。常见的工程规范（如阿里 Java 开发手册）不允许使用 Executors 的工厂方法，因为无界队列或无限线程会在任务积压时耗尽内存，导致 OOM。
