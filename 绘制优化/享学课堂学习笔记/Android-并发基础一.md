### **Android-并发基础一**



**1.进程的定义**

* 进程是程序运行资源分配的最小单位（资源包括：CPU、内存空间、磁盘 IO等）
* 同一进程中的多条线程共享该进程中的全部系统资源
* 进程是系统进行资源分配和调度的一个独立单位



**2.线程的定义**

* 线程是 CPU 调度的最小单位,必须依赖于进程而存在。
* 线程自己基本上不拥有系统资源,只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈)
* 与同属一个进程的其他的线程共享进程所拥有的全部资源
* 线程无处不在



**3.CPU 核心数和线程数的关系**

现在的电脑一般都是多核的，增加核的目的就是为了增加线程数，他们之前的关系一般情况是1：1

Intel 引入**超线程技术后**,使核心数与线程数形成 1:2 的关系。



**4.CPU 时间片轮转机制**

**时间片轮转调度**是一种最古老、最简单、最公平且使用最广的算法,又称RR调度。每个进程被分配一个时间段,称作它的时间片,即该进程允许运行的时间。

将时间片设为100ms 通常是一个比较合理的折衷。



**5.并行和并发**

比如有两个检测核酸的点，那么就可以同时有两个人在检测，那么就是并行为2。

比如只有一个核酸检测点，10分钟检测20个，那就就可以认为10分钟的并发量是20个

并发一定要加个**单位时间**,也就是说单位时间内并发量是多少，离开了单位时间其实是没有意义的。

**并发**:指应用能够交替执行不同的任务,比如单 CPU 核心下执行多线程并非是同时执行多个任务,如果你开两个线程执行,就是在你几乎不可能察觉到的速度不断去切换这两个任务

**两者区别:一个是交替执行,一个是同时执行**



**6.高并发编程的意义**

* 充分利用 CPU 的资源
* 加快响应用户的时间
* 使代码模块化、异步化、简单化

例如电商系统，下订单和给用户发送短信、邮件就可以进行拆分，将给用户发送短信、邮件这两个步骤独立为单独的模块，并交给其他线程去执行。这样既增加了异步的操作，提升了系统性能，又使程序模块化，清晰化和简单化。



**7.多线程程序需要注意事项**

* 线程之间的安全性

  在同一个进程里面的多线程是**资源共享**的,也就是都可以访问同一个内存地址当中的一个变量。

  若有多个线程同时执行写操作,一般都需要考虑线程同步,否则就可能影响线程安全。

* 线程之间的死锁

* 线程太多了会将服务器资源耗尽形成死机当机，不要开过多的线程



**8.Java 程序天生就是多线程的**

```java
public static void main(String[] args) {
    //Java 虚拟机线程系统的管理接口
    ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
    // 不需要获取同步的monitor和synchronizer信息，仅仅获取线程和线程堆栈信息
    ThreadInfo[] threadInfos =
            threadMXBean.dumpAllThreads(false, false);
    // 遍历线程信息，仅打印线程ID和线程名称信息
    for (ThreadInfo threadInfo : threadInfos) {
        System.out.println("[" + threadInfo.getThreadId() + "] "
                + threadInfo.getThreadName());
    }
}
```

输出：

[6] Monitor Ctrl-Break ///监控 Ctrl-Break 中断信号的
[5] Attach Listener //内存 dump，线程 dump，类信息统计，获取系统属性等
[4] Signal Dispatcher // 分发处理发送给 JVM 信号的线程
[3] Finalizer  //系统GC 调用对象 finalize 方法的线程
[2] Reference Handler //清除 Reference 的线程
[1] main //线程，用户程序入口

未添加任何其他的代码逻辑，jvm就已经为我们启动了6个线程，java天生就是多线程的。



**9.启动线程的方式**

There are two ways to create a new thread of execution

这个是Thread源码的官方注释上的，所以启动线程的方式有两种。



**继承Thread的方式：**

```java
private static class PersonThread extends Thread{
   @Override
   public void run() {
      super.run();
      // do somthine;
      System.out.println("I am Thread");
   }
}
//启动方式
PersonThread personThread = new PersonThread();
personThread.start();  //start 不能调用多次，否则会抛出来异常
```



**实现了Runnable接口的方式**

```java
private static class PersonRunnable implements Runnable{

   @Override
   public void run() {
      //  do somthine;
      System.out.println("I am implements Runnable");
   }
}

//启动方式：
PersonRunnable personRunnable = new PersonRunnable();
new Thread(personRunnable).start();
```

**Thread 和 Runnable 的区别：**

 Thread 才是 Java 里对线程的唯一抽象，Runnable 只是对任务（业务逻辑）的抽象。Thread 可以接受任意一个 Runnable 的实例并执行。



**10.安全的停止线程**

从Thread源码中可以看到suspend()、resume() 和 stop()这些方法都已经废弃，不再建议使用。

**suspend()**：挂起废弃的的原因是在调用后，线程不会释放已经占有的资源（比如锁），而是占有着资源进入睡眠状态，**这样容易引发死锁问题。**

 **stop()**：这个方法废弃的原因是：stop()方法在终结一个线程时不会保证线程的资源正常释放，通常是没有给予线程完成资源释放工作的机会，因此会导致程序可能工作在不确定状态下。



安全的中止则是其他线程通过调用某个线程 的 interrupt()方法对其进行中断操作,这个方法只是给了一个停止线程的标记位置，并没有真的停止线程，设置标记位置的线程可以不理会这个标记位继续执行。

也可以通过 isInterrupted()来获取标记位置，来停止线程的执行。

**Thread.interrupted()** 会将中断标识位改写为 false，并返回标记为的值，这个是一个静态方法。

```java
private static class PersonThread extends Thread{

   public PersonThread(String name) {
      super(name);
   }

   @Override
   public void run() {
      String threadName = Thread.currentThread().getName();
      System.out.println(threadName+" interrrupt flag ="+isInterrupted());
      while(!isInterrupted()){
         System.out.println(threadName+"inner interrrupt flag ="
               +isInterrupted());
      }
      System.out.println(threadName+" interrrupt flag ="+isInterrupted());
   }
}
//在main方法中执行
Thread endThread = new UseThread("endThread");
endThread.start();
Thread.sleep(10);
endThread.interrupt();//中断线程，其实设置线程的标识位true
```