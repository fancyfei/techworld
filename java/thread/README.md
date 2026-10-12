# Java 线程专题

覆盖从 JVM 线程模型、内存模型，到锁机制、线程安全、线程池的完整知识链。

- [thread](thread.md)   JVM 线程架构。    1:1 线程模型与三层结构、虚拟线程、Java 线程状态与 OS 状态映射、调度与上下文切换开销、JVM 内部线程 

- [thread_mem_model](thread_mem_model.md)  Java 内存模型。  主内存与工作内存、8 种内存操作及其规则、volatile 特殊规则、synchronized 的内存语义、happens-before 规则 

- [thread_lock](thread_lock.md)  锁机制与实现。  锁的基本概念、悲观锁与乐观锁、编译器锁优化（锁消除/锁粗化）、自旋锁、公平与非公平锁、锁状态升级、可重入锁、独占锁与共享锁、AQS、对象头与 Monitor 

- [thread_safe](thread_safe.md)  线程安全。  线程安全的定义、原子性/可见性/有序性、线程安全程度五级分类、互斥同步（synchronized/ReentrantLock）、非阻塞同步（CAS）、代码级线程安全（可重入代码/线程本地存储） 

- [thread_pool](thread_pool.md)  线程池。  池化思想、Executor 框架与 ThreadPoolExecutor、生命周期五态、核心参数、阻塞队列、拒绝策略、Executors 工厂方法 

