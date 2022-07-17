### Android性能优化一代码优化

代码层面优化是最基本的能力，每个研发人员都应该努力提高自己的编码能力，从而写出高效的代码。

编写代码高效代码要遵循两个准则

* 不要做冗余的工作
* 避免次数过多的内存分配操作



**1.数据结构的选择**

ArrayList与LinkedList



HashMap与HashSet



SparseArray代替HashMap

SparseArray是Android平台特有的稀疏数组，它是Integer到Object的一个映射，在特定的场合可以提高效率，它的核心是二分查找算法。

SparseArray有一下几种：

* SparseBooleanArray 可替代HashMap<Integer,Boolean>
* SparseIntArray 可替代HashMap<Integer,Integer>
* SparseLongArray 可替代HashMap<Integer,Long>
* SparseStringArray 可替代HashMap<Integer,String>



注意：SparseArray不是线程安全的。

​			由于要进行二分查找，因此，SparseArray会按照插入数据的key值大小顺序插入

​			SparseArray对删除做了优化，并不会立即删除这个元素，而是通过设置标识位的方式，后面尝试重用。

​			这个替换优化可以借助Lint检查的提示来对项目进行优化。



**2.Handler和内部类的正确写法**

进程间通信会进程用到Handler，这个Handler如果写法不规范也很容易出现内存泄漏。但是，为什么会产生内存泄漏，很多人可能将不清楚。分析一下产生内存泄漏的原因：

Handler是和Looper以及MessageQueue一起工作的，在Android中一个应用启动后，系统默认会创建一个会主线程服务的Looper，该Looper处理主线程所有的Message消息，它的生命周期贯穿整个应用的生命周期。在主线程使用Handler都会默认绑定这个Looper对象，在主线程创建Handler对象会马上关联主线程的Looper的MessageQueue，这时发送到MessageQueue的Message都持有这个Handler的对象的引用，这样在Looper处理消息的时候才能回调到Handler的HandleMessage方法，因此，如果Message消息还没有被处理完，那么Handler对象也就不会被垃圾回收。

在java中，**非静态匿名内部类会持有外部类的一个隐式引用**，这样就可能导致外部类无法被垃圾回收。

Handler的正确写法

```java
private final InnerHandler mInnerHandler = new InnerHandler(this);

//静态内部类，持有外部类的弱引用。
private static class InnerHandler extends Handler{
    
    private final WeakReference<HandlerActivity> weakReference;

    private InnerHandler(HandlerActivity handlerActivity) {
        this.weakReference = new WeakReference<HandlerActivity>(handlerActivity);
    }

    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);
    }
}
```

静态内部类，持有外部类的弱引用。

静态的匿名内部类不持有外部类的引用。



**3.正确使用Context**



