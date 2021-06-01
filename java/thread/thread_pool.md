# Java 线程池

现在主流CPU都是多核的，使用多线程并行计算提升服务器性能是程序的基本手段。但是操作系统频繁创建线程和销毁线程需要时间，出现了基于池化思想管理线程的工具，线程池。

- 线程池目的：执行提交给线程池的任务，执行完一个任务后不会退出，而是继续等待或执行新任务。
- 线程池的组成：一个是任务队列，另一个是工作者线程，任务队列保存待执行的任务，工作者线程循环从队列中接受任务并执行。

## 线程池的好处

- 降低系统资源消耗，通过重用降低线程创建和销毁造成的消耗；
- 提高系统响应速度，任务复用已有线程，少了创建过程；
- 方便线程并发数的管控，可以限制线程池中线程的数量。
- 减少负载高峰冲击，排队处理任务，可消峰填谷，一定程度的防止系统过载。
- 提供更强大的功能，如延时定时等。

 ## Java的Executor

Java并发包中提供了线程池管理类Executor，它提供了一种思想：将任务提交和任务执行进行解耦。把Runnable对象的任务，提交给Executor框架完成线程的调配和执行。

线程池中是以生产者消费者模式，通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。

Executor的实现类是ThreadPoolExecutor类，线程池内部使用一个变量维护两个值：运行状态(runState)和线程数量 (workerCount)。变量是AtomicInteger类型，使用不同的位来表示两个值。

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
//核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
private volatile int   corePoolSize;     
//线程池最大能容忍的线程数
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

- RUNNING，能够接收新任务，以及对已添加的任务进行处理。
- SHUTDOWN，不接收新任务，但能处理已添加的任务。
- STOP，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。
- TIDYING，当所有的任务已终止，变为此状态后会调钩子函数terminated()。
- TERMINATED，线程池彻底终止。

## 线程池的重要的参数

- corePoolSize（核心线程池大小），正常能创建的最大线程数。大于此数的多出来的任务放缓存队列中。
- maximumPoolSize（线程池最大大小），当核心线程和队列都满了，还能创建出的最大线程数。
- keepAliveTime（线程存活保持时间），那些大于核心线程数时创建出的线程空闲的最大时间。
- workQueue（任务队列），用于传输和保存等待执行任务的阻塞队列。
- threadFactory（线程工厂），用于创建新线程。
- handler（任务拒绝策略），当线程池和队列都满了，新任务会执行的策略。

## 阻塞队列种类

参数workQueue的类型是阻塞队列BlockingQueue，常见的可以用作线程池队列的有：

- ArrayBlockingQueue，基于数组的有界阻塞队列。
- LinkedBlockingQueue，基于链表的阻塞队列，可以指定最大长度，但默认是无界的。
- SynchronousQueue，没有实际存储空间的同步阻塞队列。

线程池的任务排队拒绝策略与BlockingQueue是有关联的。

## 拒绝策略

拒绝策略是可以自定义的，ThreadPoolExecutor已经实现了四种处理方式：

- ThreadPoolExecutor.AbortPolicy：这就是默认的方式，直接抛出异常。
- ThreadPoolExecutor.DiscardPolicy：静默处理，忽略新任务，不抛异常，也不执行。
- ThreadPoolExecutor.DiscardOldestPolicy：将等待时间最长的任务扔掉，然后自己排队。
- ThreadPoolExecutor.CallerRunsPolicy：在任务提交者线程中执行任务，而不是交给线程池中的线程执行。

## Executors实现的线程池

Executors ，一个简单的类可以让你创建常见场景的线程池和线程工厂。Executors 可以创建以下六种线程池。

- FixedThreadPool(n)：创建一个数量固定的线程池，超出的任务排队等待，可用于控制程序的最大并发数。最大可创建Integer.MAX_VALUE个线程，缓冲任务的队列为LinkedBlockingQueue。
- CachedThreadPool()：短时间内处理大量工作的线程池，会根据任务数量产生对应的线程，并试图缓存线程以便重复使用，最大存活60秒。最大可创建Integer.MAX_VALUE个线程，缓冲任务的队列为SynchronousQueue。
- SingleThreadExecutor()：创建一个单线程线程池。其它的任务都放在LinkedBlockingQueue中排队等待执行。
- ScheduledThreadPool(n)：创建一个数量固定的线程池，支持执行定时性或周期性任务。缓冲任务的队列为DelayedWorkQueue。最大可创建Integer.MAX_VALUE个线程。
- SingleThreadScheduledExecutor()：此线程池就是单线程的 newScheduledThreadPool。
- WorkStealingPool(n)：Java 8 新增创建线程池的方法，创建抢占式并行线程池，不能保证执行顺序。

