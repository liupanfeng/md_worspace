# Android-并发基础二



**1.线程启动的方式**

启动线程的方式只有两种：

* 继承Thread，实例化，调用start方法
* 实现Runnable接口，并交给Thread去执行



**2.线程状态**

java中线程有2种状态

* 初始(NEW)：新创建了一个线程对象，但还没有调用start()方法。
* 运行(RUNNABLE)：Java 线程中将**就绪（ready）**和**运行中（running）**两种状态笼统的称为“运行”。
  * CPU未给分配时间片的状态称为就绪状态
  * 拿到CPU的时间片称为运行中
* 阻塞(BLOCKED)：表示线程阻塞于锁。
* 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
* 超时等待(TIMED_WAITING)：该状态不同于 WAITING，它可以在指定的时间后自行返回。
* 终止(TERMINATED)：表示该线程已经执行完毕。



**3.死锁**

**死锁是指**：两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁。

死锁产生的四个必要条件：

* **互斥条件**：指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的线程用毕释放。
* **请求和保持**：指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。
* **不剥夺条件**：指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
* **环路等待**：指在发生死锁时，必然存在一个进程——资源的环形链。



模拟一个死锁的场景如下：

```java
private static Object mLock1 = new Object();//第一个锁
private static Object mLock2 = new Object();//第二个锁

/**
 * 先拿到锁1 再申请锁2
 * @throws InterruptedException
 */
private static void task1() throws InterruptedException {
    String threadName = Thread.currentThread().getName();
    synchronized (mLock1){
        System.out.println(threadName+" get lock 1");
        Thread.sleep(100);
        synchronized (mLock2){
            System.out.println(threadName+" get lock 2");
        }
    }
}

/**
 * 先申请锁2 再申请锁1
 * @throws InterruptedException
 */
private static void task2() throws InterruptedException {
    String threadName = Thread.currentThread().getName();
    synchronized (mLock2){
        System.out.println(threadName+" get lock 2");
        Thread.sleep(100);
        synchronized (mLock1){
            System.out.println(threadName+" get lock 1");
        }
    }
}

private static class TestThread extends Thread{

    private String name;

    public TestThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        Thread.currentThread().setName(name);
        try {
            task1();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public static void main(String[] args) throws InterruptedException {
    Thread.currentThread().setName("main thread");
     //子线程执行任务1
    TestThread testThread = new TestThread("test thread");
    testThread.start();
    //主线程执行任务2
    task2();
}
```

* 申明了两个锁，锁1、锁2
* 有两个task任务，任务1先申请锁1，然后再申请锁2，任务2先申请锁2，再申请锁1
* 子线程执行任务1、主线程执行任务2

这样就产生了，死锁。



产生的影响是：

* 线程不工作了，但是整个程序还是活着的
* 没有任何的异常信息可以供我们检查
* 一旦程序发生了发生了死锁，是没有任何的办法恢复的，只能重启程序，对正式已发布程序来说，这是个很严重的问题。



**那么如何解决死锁呢？**

* 内部通过顺序比较，确定拿锁的顺序，比如都是先拿锁1再拿锁2
* 采用尝试拿锁的机制：tryLock 



**4.活锁**

两个线程在尝试拿锁的机制中，发生多个线程之间互相谦让，不断发生同一个线程总是拿到同一把锁，在尝试拿另一把锁时因为拿不到，而将本来已经持有的锁释放的过程。 

解决办法：每个线程休眠随机数，错开拿锁的时间。



**5.线程饥饿**

由于线程的优先级比较低，一直无法获取到CPU分配的时间片进行执行，这个现象被称为线程饥饿。



**6.ThreadLocal**

ThreadLocal 和 Synchonized 都用于解决多线程并发访问问题。

可是ThreadLocal 与 synchronized 有本质的差别。**synchronized 是利用锁的机制**，使变量或代码块在某一时该仅仅能被一个线程访问。而 **ThreadLocal 为每个线程都提供了变量的副本**，使得每个线程在某一时间访问到的并非同一个对象，这样就隔离了多个线程对数据的数据共享。



**ThreadLocal的使用：**

ThreadLocal 类接口很简单，只有 4 个方法：

*  void set(Object value)： 设置当前线程的线程局部变量的值。
* public Object get()：该方法返回当前线程所对应的线程局部变量。
* public void remove()：将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK5.0 新增的方法。需要指出的是，**当线程结束后，对应该线程的局部变量将自动被垃圾回收**，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。
* protected Object initialValue()：返回该线程局部变量的初始值，该方法是一个 protected 的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1 次调用get() 或 set(Object)时才执行，并且仅执行 1 次。ThreadLocal 中的缺省实现直接返回一个 null。

**public final static ThreadLocal RESOURCE = new ThreadLocal();RESOURCE代表一个能够存放String类型的ThreadLocal对象。此时不论什么一个线程能够并发访问这个变量，对它进行写入、读取操作，都是线程安全的。**



**持有关系是：**

从下面的源码中我们可以看到：Thread持有ThreadLocalMap，这个map里边持有一个Entry对象数组，Entry的key就是ThreadLocal，value就是需要保存的数据。

```java
public class Thread implements Runnable {
    
    ...
        
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}
```

线程持有ThreadLocalMap成员变量



```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * The initial capacity -- MUST be a power of two.
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;
```

ThreadLocalMap持有Entry数组表table成员变量

Entry的key保存的就是ThreadLocal对象，使用弱引用进行存储的，value就是我们保存的Object数据。



**7.CAS基本原理**

**CAS**：Compare and swap，比较并交换，实现原子操作的一种方式。

**CAS基本思路**：每一个 CAS 操作过程都包含三个运算符：一个内存地址 V，一个期望的值 A 和一个新值 B，操作的时候如果这个地址上存放的值等于这个期望的值 A，则将地址上的值赋为新值 B，否则不做任何操作。

get获取变量的值（旧值）--》计算后得到新值--》Compare内存中的变量值跟旧值进行比较，如果相等--》将旧值更改为新值，如果Compare比较不相等，再重来一次，这就是CAS的执行思路。



**源码中Atomic开头的类都是原子操作类**：

* **AtomicInteger**

  * int addAndGet（int delta）：以原子方式将输入的数值与实例中的值（AtomicInteger 里的 value）相加，并返回结果。
  * boolean compareAndSet（int expect，int update）：如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。
  * int getAndIncrement()：以原子方式将当前值加 1，注意，这里返回的是自增前的值。
  * int getAndSet（int newValue）：以原子方式设置为newValue 的值，并返回旧值

* **AtomicIntegerArray**

  * int addAndGet（int i，int delta）：以原子方式将输入值与数组中索引i 的元素相加。
  * boolean compareAndSet（int i，int expect，int update）：如果当前值等于预期值，则以原子方式将数组位置 i 的元素设置成 update 值。
  * 需要注意的是:数组 value 通过构造方法传递进去，然后AtomicIntegerArray会将当前数组复制一份，所以当 AtomicIntegerArray 对内部的数组元素进行修改时，不会影响传入的数组。

* **AtomicBoolean**

* **AtomicLong**

* **AtomicReference**:原子更新引用类型,如果有多个变量需要进行原子操作，可以封装到一个对应里边使用原子引用类型。

* **AtomicStampedReference**：利用版本戳的形式记录了每次改变以后的版本号，这样的话就不会存在ABA问题了，这就是 AtomicStampedReference 的解决方案。**AtomicStampedReference 是使用pair 的int stamp 作为计数器使用**，AtomicStampedReference 可以得到数据被修改过几次。

* **AtomicMarkableReference**：原子更新带有标记位的引用类型。可以原子更新一个布尔类型的标记位和引用类型。这个类只关心是否被修改过，修改几次不管。

  public AtomicMarkableReference(V initialRef, boolean initialMark) {
      pair = Pair.of(initialRef, initialMark);
  }



**使用CAS存在的问题：**

* **ABA问题**：因为 CAS 需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是 A，变成了 B，又变成了A，那么使用CAS 进行检查时会发现它的值没有发生变化，但是实际上却变化了。**解决办法：使用AtomicStampedReference或者AtomicMarkableReference**。

* **循环时间长开销大**：自旋 CAS 如果长时间不成功，会给 CPU 带来非常大的执行开销。

* **只能保证一个共享变量的原子操作**：当对一个共享变量执行操作时，我们可以使用循环CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁。还有一个办法，就是把多个共享变量合并成一个共享变量来操作。



**8.阻塞队列**

**队列**：是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。

**阻塞队列**：

* 支持阻塞的插入方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。
* 支持阻塞的移除方法：意思是在队列为空时，获取元素的线程会等待队列变为非空。

**并发编程**中使用**生产者和消费者模式**能够解决绝大多数并发问题。该模式通过平衡生产线程和消费线程的工作能力来提高程序整体处理数据的速度。

为了解决这种生产消费能力不均衡的问题，便有了生产者和消费者模式。

生产者和消费者模式是通过一个容器来解决生产者和消费者的强耦合问题

生产者和消费者彼此之间不直接通信，而是**通过阻塞队列来进行通信**，所以生产者生产完数据之后不用等待消费者处理，直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列里取，**阻塞队列就相当于一个缓冲区**，平衡了生产者和消费者的处理能力。



**阻塞队列里边的方法也不全是阻塞的，可以分为四类：**

* **抛出异常**：当队列满时，如果再往队列里插入元素，会抛出IllegalStateException（"Queuefull"）异常。当队列空时，从队列里获取元素会抛出 NoSuchElementException 异常。
  * add()
  * remove() 
  * elment()  
* **返回特殊值**：当往队列插入元素时，会返回元素是否插入成功，成功返回true。如果是移除方法，则是从队列里取出一个元素，如果没有则返回null。
  * offer()
  * poll()
  * peek()
* **一直阻塞**：当阻塞队列满时，如果生产者线程往队列里put 元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者线程从队列里 take 元素，队列会阻塞住消费者线程，直到队列不为空。
  * put()
  * take()
* **超时退出**：当阻塞队列满时，如果生产者线程往队列里插入元素，队列会阻塞生产者线程一段时间，如果超过了指定的时间，生产者线程就会退出。
  * offer(e,time,unit)
  * poll(time,unit)



常见的阻塞队列有：

* **ArrayBlockingQueue**：一个由数组结构组成的有界阻塞队列。
* **LinkedBlockingQueue**：一个由链表结构组成的有界阻塞队列。
* PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。
* **DelayQueue**：一个使用优先级队列实现的无界阻塞队列。
* SynchronousQueue：一个不存储元素的阻塞队列。
* LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
* LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

以上的阻塞队列都实现了 BlockingQueue 接口，也都是线程安全的。颜色加深的三个比较常用。



