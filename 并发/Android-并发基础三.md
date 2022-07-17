# Android-并发基础三

[TOC]

#### 1.线程池

##### 1.1为什么需要使用线程池？

* 降低资源消耗：通过服用已经存在的线程，降低创建线程和销毁线程对资源的消耗。

* 提高相应速度：任务来了，不需要等待线程的创建，直接使用线程执行任务。

* 提高线程的可管理性：线程是稀缺资源，如果无限制的创建，不仅会消耗CPU的资源，还会降低系统的稳定性，使用线程池可以统一分配、调优和监控线程。

  

##### 1.2线程池相关的类分析：

**Executor**：是一个接口，是Executor架构的基础，是用来将任务的提交和任务的执行进行分离。

**ExsecutorService**：继承Executor，增加了shutdown()、submit()，可以认为是线程池的真正的接口。

**AbstractExecutorService**：实现了ExecutorService中的大部分方法。

**ThreadPoolExecutor**：是线程池的核心类，用来执行被提交的任务。

**ScheduledExecutorService**：继承了ExecutorService，提供了带周期执行功能的ExecutorService。

**ScheduledThreadPoolExecutor**：是一个实现类，可以在给定的延迟后运行命令或则定期执行，比Timer灵活，功能强大。



##### 1.3线程池的构造方法：

```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
    this.ctl = new AtomicInteger(ctlOf(-536870912, 0));
    this.mainLock = new ReentrantLock();
    this.workers = new HashSet();
    this.termination = this.mainLock.newCondition();
    if (corePoolSize >= 0 && maximumPoolSize > 0 && maximumPoolSize >= corePoolSize && keepAliveTime >= 0L) {
        if (workQueue != null && threadFactory != null && handler != null) {
            this.corePoolSize = corePoolSize;
            this.maximumPoolSize = maximumPoolSize;
            this.workQueue = workQueue;
            this.keepAliveTime = unit.toNanos(keepAliveTime);
            this.threadFactory = threadFactory;
            this.handler = handler;
        } else {
            throw new NullPointerException();
        }
    } else {
        throw new IllegalArgumentException();
    }
}
```

* **corePoolSize**：核心线程数量，当提交一个任务时，线程池会创建一个新的线程执行任务直到线程数量等于核心线程数量。如果再继续提交任务，任务会添加到阻塞队列，并不会再创建新的线程。如果执行prestartAllCoreThreads()这个方法会提前创建并启动所有的核心线程数量。
* **maximumPoolSize**：当阻塞队列也满了，再提交任务就会创建新的线程执行，前提是线程的数量小于maximumPoolSize的数量。
* **keepAliveTime**：线程空闲的时候，线程的存活时间。默认情况下，只有当线程数量大于核心线程数量才会有效。
* **TimeUnit**：keepAliveTime的时间单位。
* **BlockingQueue<Runnable>**：必须是BlockingQueue阻塞队列才行。当线程池的数量大于corePoolSize数量，线程会进入阻塞队列进行阻塞等待，这样线程池也实现了阻塞的功能。一般来说，应该使用有界阻塞队列，避免将资源耗尽的风险。
* **ThreadFactory**：创建现成的线程工厂，可以统一给线程设置名称等。
* **RejectedExecutionHandler**：线程池饱和策略，当阻塞队列满了，切没有空闲线程的时候，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了四种策略：
  * **AbortPolicy**：直接抛出异常，默认策略。
  * **CallerRunsPolicy**：使用调用线程来执行这个任务。谁调用谁执行。
  * **DiscardOldestPolicy**：丢弃队列最靠前的任务，并执行当前任务。
  * **DiscardPolicy**：直接丢弃任务。



##### 1.4线程池的工作机制

* 如果当前运行的Thread小于核心线程数量，直接创建新的线程执行任务。
* 如果线程数量等于corePoolSize，则会将新添加的Tast给添加到阻塞队列中。
* 如果阻塞队列满了，那提交的任务会创建新的线程执行任务。
* 当线程数量大于maximumPoolSize，任务将被拒绝，并抛出异常。



##### 1.5线程池任务的提交

* execute():用于提交不需要返回值的任务，无法判断任务是否执行完毕等。
* submit():用于提交需要返回值的任务，线程池会返回一个Future对象，通过这个对象可以判断任务是否执行成功，并通过Future的get方法来得到返回值，get方法会阻塞当前线程知道执行完毕为止，也可以阻塞一段时间到时间立即返回，此时任务可能并未执行完毕。



##### 1.6线程池的关闭

可以通过shutdown和shutdownNow来关闭线程池，原理都是遍历线程池中的工作线程，然后逐个调用interrupt来中断线程，所以无法响应中断任务的线程可能永远无法停止。

* shutdown()：只是将线程池的状体置为SHUTDOWN状态，然后终止所以没有在执行任务的线程。
* shutdownNow()：首先将线程池的状态置为Stop，然后尝试停止所有正在执行任务的线程，并返回正在执行任务的列表。

注意只要调用上面的任何一个方法，isShutDown就会返回true，但是只要所有的任务都已经关闭，线程池才会真正关闭。此时调用isTerminated()会返回true，通常调用shutdown来关闭线程池，如果不关心任务是否执行完，也可以调用shutdownNow方法。



##### 1.7合理位置线程池

配置线程池需要分析任务的类型：

* 任务的性质：CPU密集型还是IO密集型
* 执行任务的优先级：是否又明确的优先级。
* 执行任务的时间：任务是否明确可以估计大约的耗时。
* 执行的任务是否又依赖：如是否需要连接数据库。

CPU密集型应该配置较小的线程数量，一般是Ncpu+1个线程的线程池。

IO密集型线程不一定在执行任务，所以可以多分配线程如：2*Ncpu

如果有任务有优先级，可以使用优先级队列priorityBlockQueue()



#### 2.AQS

队列同步器：AbstractQueuedSynchronizer，是用来构建锁和其他同步器的基础架构，它是用一个int类型的值来表示同步状态，同步内置的FIFO队列来完成资源获取线程的排队工作，设计者期望它称为实现大部分的同步操作的基础。

























