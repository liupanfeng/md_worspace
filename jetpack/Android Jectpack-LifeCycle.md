### Android Jetpack-Lifecycle

Android标准化项目架构：MVVM+Jectpack

助力研发，本篇将对Jectpack 中的Lifecycle进行简要分析



**1.什么是Lifecycle**

Lifecycle是Jatpack里的一个组件，可以监听Activity、Fragment声明周期的各种变化。

Lifecycle介绍：

* **可以有效的避免内存泄漏和解决Android声明周期的不方便监听的问题。**
* 是一个表示Android声明周期以及状态的对象。
* LivecycleOwner 用于连接有声明周期的对象，如Activity、fragment。
* LivecycleObserveder 用于观察LivecycleOwner，LivecycleOwner是被观察者。
* Lifecycle原理是通过使用观察者监听被观察者的声明周期变化实现的。

使用这个可以控制在界面可见的时候才会刷新页面，页面不可见停止页面刷新。



**2.通过继承LifecycleObserver实现声明周期感知**

```java
class MSObserverListener : LifecycleObserver {

     companion object{
          const val TAG:String="MSLocationListener";
     }

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun create() = Log.d(TAG, "create----")

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun start() = Log.d(TAG, "start---- ")

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun resume() = Log.d(TAG, "resume---- ")

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun pause() = Log.d(TAG, "pause---- ")

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun stop() = Log.d(TAG, "stop----")

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun destroy() = Log.d(TAG, "destroy---- ")

}

class MainActivity : AppCompatActivity() {

    companion object{
        const val TAG="MainActivity";
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
            
        lifecycle.addObserver(MSObserverListener())
            
    }
}
```

通过addObserver 建立观察订阅关系。



**3.通过继承DefaultLifecycleObserver**

```kotlin
class MSObserver: DefaultLifecycleObserver{

    companion object{
        const val TAG:String="MSObserver";
    }

    override fun onCreate(owner: LifecycleOwner) {
        super.onCreate(owner)
        Log.d(TAG, "create---- 正在启动系统定位服务中...")
    }

    override fun onStart(owner: LifecycleOwner) {
        super.onStart(owner)
        Log.d(TAG, "start----")
    }

    override fun onResume(owner: LifecycleOwner) {
        super.onResume(owner)
        Log.d(TAG, "resume---- ")
    }

    override fun onPause(owner: LifecycleOwner) {
        super.onPause(owner)
        Log.d(TAG, "pause----")
    }

    override fun onStop(owner: LifecycleOwner) {
        super.onStop(owner)
        Log.d(TAG, "stop---- ")
    }

    override fun onDestroy(owner: LifecycleOwner) {
        super.onDestroy(owner)
        Log.d(TAG, "destroy----")
    }
}

class MainActivity : AppCompatActivity() {

    companion object{
        const val TAG="MainActivity";
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        lifecycle.addObserver(MSObserver())
    }
}
```

* 通过实现DefaultLifecycleObserver 来实现，可以拿到 Activity Fragment 所有环境
* 可以在观察者里边执行一些跟上下文相关的代码逻辑，比如弹出Toast等

这个方式更常用一些。



**4.通过内联函数实现**

```kotlin
class MainActivity : AppCompatActivity() {

    companion object{
        const val TAG="MainActivity";
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
       lifecycle.addObserver(MSInnerObserver())

    }

    /**
     * 通过内联函数实现，感知声明周期变化
     */
    inner class MSInnerObserver:LifecycleObserver{
        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        fun onResume(){
           Log.d(TAG,"inner onResume-----")
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        fun onPause(){
            Log.d(TAG,"inner onPause-----")
        }
    }
}
```

通过内联函数实现声明周期的感知，可以实现代码逻辑内聚的效果。

一般在实际项目中，创建订阅关系会在BaseActivity



