### Android Jetpack-LiveData

Android标准化项目架构：MVVM+Jectpack

助力研发，本篇将对Jectpack 中的LiveData进行简要分析



**1.LiveData是什么？**

LiveData是一种可观察的数据存储类。LiveData具有声明周期感知能力，这种能力可以确保LiveData的更新只会存在于活跃的声明周期状态的应用组件观察者。



**2.定义一个LiveData**

```kotlin
object MSLiveData {
 /**
  * 懒加载，用的时候再加载
  */
 val info:MutableLiveData<String> by lazy {
        MutableLiveData()
    }

}
```

```
class LiveDataActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)
        val content: TextView =findViewById(R.id.content)
        MSLiveData.info.observe(this) {
            content.text = it
        }
        MSLiveData.info.value="唐三"
    }


}
```

在Activity添加observe，在回调里边更新UI，只要MSLiveData.info数据发生变化就会更新UI



**3.在子线程更改数据**

```kotlin
/**
 * 模拟后台发送数据
 */
class MSService : Service() {

    override fun onBind(intent: Intent): IBinder ? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        thread {
            for (x in 1..50000) {
                Log.d("server", "服务器给推你推送消息,消息内容是:${x}")
                MSLiveData.info.postValue("服务器给推你推送消息啦,消息内容是:${x}")
                Thread.sleep(3000) // 3秒钟推一次
            }
        }
        return super.onStartCommand(intent, flags, startId)
    }
}
```

子线程模拟服务器发送推送数据，数据会持续发送。

```Kotlin
class LiveDataActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)
        val content: TextView =findViewById(R.id.content)
        val btnStartService: Button =findViewById(R.id.button)

        btnStartService.setOnClickListener{
            startService(Intent(this,MSService::class.java))
            Toast.makeText(LiveDataActivity@this,"启动服务后台模拟后台推送数据",Toast.LENGTH_LONG).show()
        }

        MSLiveData.info.observe(this) {
            content.text = it
            Log.d("server", "界面可见，更新消息列表UI界面:${it}")
        }

    }


}
```

分别在页面可见以及推到后台，页面的刷新情况以及日志的打印

日志输出：

```shell
2022-05-03 16:26:38.568 7836-7836/com.meishe.jetpackcollection D/server: 界面可见，更新消息列表UI界面:服务器给推你推送消息啦,消息内容是:2
2022-05-03 16:26:41.567 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:3
2022-05-03 16:26:41.569 7836-7836/com.meishe.jetpackcollection D/server: 界面可见，更新消息列表UI界面:服务器给推你推送消息啦,消息内容是:3
2022-05-03 16:26:44.568 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:4
2022-05-03 16:26:44.570 7836-7836/com.meishe.jetpackcollection D/server: 界面可见，更新消息列表UI界面:服务器给推你推送消息啦,消息内容是:4
2022-05-03 16:26:46.887 7836-7836/com.meishe.jetpackcollection D/MSObserver: pause----
2022-05-03 16:26:47.350 7836-7836/com.meishe.jetpackcollection D/MSObserver: stop---- 
2022-05-03 16:26:47.570 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:5
2022-05-03 16:26:50.576 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:6
2022-05-03 16:26:53.582 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:7
2022-05-03 16:26:54.711 7836-7836/com.meishe.jetpackcollection D/MSObserver: start----
2022-05-03 16:26:54.711 7836-7836/com.meishe.jetpackcollection D/server: 界面可见，更新消息列表UI界面:服务器给推你推送消息啦,消息内容是:7
2022-05-03 16:26:54.712 7836-7836/com.meishe.jetpackcollection D/MSObserver: resume---- 
2022-05-03 16:26:56.588 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:8
2022-05-03 16:26:56.589 7836-7836/com.meishe.jetpackcollection D/server: 界面可见，更新消息列表UI界面:服务器给推你推送消息啦,消息内容是:8
2022-05-03 16:26:59.589 7836-7900/com.meishe.jetpackcollection D/server: 服务器给推你推送消息,消息内容是:9
```

当页面不可见之后，数据在发送，但是livedata的回调方法不会继续执行，**这样就保证了页面可见的时候数据才会刷新。**



**4.LiveData数据黏性**

```kotlin
class LiveDataActivity : BaseActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)
        val startToOtherActivity: Button =findViewById(R.id.startToOtherActivity)
		
        //跳转到另外一个Activity，在跳转之前先设置旧数据
        startToOtherActivity.setOnClickListener{

            MSLiveData.info.value = "飞跃，跃然纸上1" // 以前的旧数据
            MSLiveData.info.value = "飞跃，跃然纸上2"
            MSLiveData.info.value = "飞跃，跃然纸上3"

            startActivity(Intent(this,OtherLiveDataActivity::class.java))
        }

    }


}
```

跳转到另外一个Activity，在跳转之前先设置旧数据

数据设置了3条，但是此时另外一个Activity还未进行绑定，验证是否可以收到旧数据。

```kotlin
class OtherLiveDataActivity : BaseActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_other_live_data)
        val content:TextView=findViewById(R.id.content)

        MSLiveData.info.observe(this){
            Log.d("lpf",it)
            content.text=it
        }

    }
}
```

日志输出：

```shell
2022-05-03 16:32:45.774 7836-7836/com.meishe.jetpackcollection D/lpf: 飞跃，跃然纸上3
```

可见由于数据黏性的接受到旧数据，而且是只打印了最近的一次数据。这个情况有可能导致异常。

在源码中可见，在回调数据之前有这个判断：

```java
if (observer.mLastVersion >= mVersion) {
    return;
}
observer.mLastVersion = mVersion;
observer.mObserver.onChanged((T) mData);
```

在回调数据之前，由于调用了setValue方法，mVersion做了自增，但是新绑定的mLastVersion=-1，导致判断条件直接忽略，进行了一次数据回调导致的。

```java
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```



**5.利用反射去除数据粘性**

```kotlin
/**
 * 通过反射 去掉黏性事件 
 */
object MSLiveDataKT {

    /**
     * 存放订阅者，增加缓存，为了提升性能在源码中也随处可见各种缓存
     */
    private val bus : MutableMap<String, BusMutableLiveData<Any>> by lazy { HashMap() }

    /**
     * 暴露一个函数，给外界注册
     */
    @Synchronized
    fun <T> with(key: String, type: Class<T>, isStick: Boolean = true) : BusMutableLiveData<T> {
        if (!bus.containsKey(key)) {
            bus[key] = BusMutableLiveData(isStick)
        }
        return bus[key] as BusMutableLiveData<T>
    }

    class BusMutableLiveData<T> private constructor() : MutableLiveData<T>() {

        var isStick: Boolean = false

        /**
         *  次构造 
         *  isStick：默认是true 开启数据黏性
         *  传false就执行hook操作，去除黏性
         */
        constructor(isStick: Boolean) : this() {
            this.isStick = isStick
        }

     
        override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
            super.observe(owner, observer) // 启用系统的功能
            if (!isStick) {
                Log.d("lpf","执行hook");
                executeHook(observer = observer)
            }
        }

        private fun executeHook(observer: Observer<in T>) {
            // 1.得到mLastVersion
            // 获取到LivData的类中的mObservers对象
            val liveDataClass = LiveData::class.java

            val mObserversField: Field = liveDataClass.getDeclaredField("mObservers")
            mObserversField.isAccessible = true // 私有修饰也可以访问

            // 获取到这个成员变量的对象  Any == Object
            val mObserversObject: Any = mObserversField.get(this)

            // 得到map对象的class对象   private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            //* 星投影KT   泛型的？
            val mObserversClass: Class<*> = mObserversObject.javaClass

            // 获取到mObservers对象的get方法   protected Entry<K, V> get(K k) {
            val get: Method = mObserversClass.getDeclaredMethod("get", Any::class.java)
            get.isAccessible = true // 私有修饰也可以访问

            // 执行get方法
            val invokeEntry: Any = get.invoke(mObserversObject, observer)

            // 取到entry中的value   is "AAA" is String    is是判断类型 as是转换类型
            // Any? 是 Object , 类比 Java ?
            var observerWraper: Any? = null
            if (invokeEntry != null && invokeEntry is Map.Entry<*, *>) {
                observerWraper = invokeEntry.value
            }
            if (observerWraper == null) {
                throw NullPointerException("observerWrapper is null")
            }

            // 得到observerWraperr的类对象
            val supperClass: Class<*> = observerWraper.javaClass.superclass
            val mLastVersion: Field = supperClass.getDeclaredField("mLastVersion")
            mLastVersion.isAccessible = true

            // 2.得到mVersion
            val mVersion: Field = liveDataClass.getDeclaredField("mVersion")
            mVersion.isAccessible = true

            //3.mLastVersion=mVersion
            val mVersionValue: Any = mVersion.get(this)
            mLastVersion.set(observerWraper, mVersionValue)
        }
    }

}
```

模拟使用过程，可以继续使用前面页面跳转的例子就可以。



```java
   startToOtherActivity.setOnClickListener{
			//以前的旧数据
            MSLiveDataKT.with("info", String::class.java,false).value = "飞跃，跃然纸上3"   

            startActivity(Intent(this,OtherLiveDataActivity::class.java))
        }

class OtherLiveDataActivity : BaseActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_other_live_data)
        val content:TextView=findViewById(R.id.content)

        MSLiveDataKT.with("info", String::class.java).observe(this){
            Log.d("lpf",it)
            content.text=it
        }

        Handler().postDelayed(Runnable {
            MSLiveDataKT.with("info", String::class.java).value = "新数据来啦"
        },2000)

    }
}
```



![image-20220503172126115](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220503172126115.png)

通过测试结果可以看到，旧的数据就没有了。