### Android Kotlin-协程

[T]

#### 1.线程与协程

* 线程的封装框架，从宏观角度看，可以这么认为
* 协程有点像轻量级的线程
* 从包含关系上看，协程跟线程的关系，有点像“线程与进程的关系”，协程不可能脱离线程运行
* 协程虽然不能脱离线程，但是可以在不同的线程切换

协程高效、轻量级，可以解决大并发的问题，但是协程的真正魅力是，最大的成都简化异步并发任务，**用同步代码写出异步的效果**。

#### 2.异步与协程

##### 2.1传统的方式完成异步任务网络加载

```kotlin
mProgressDialog = ProgressDialog(this)
mProgressDialog.setTitle("请求服务器中...")
mProgressDialog.show()
mTextView.setText("请求服务器中...")
thread {
    var call = RetrofitClient.instance.initRetrofit(WanAndroidAPI::class.java)
        .login("lpf666", "123456")
    val result = call.execute()


    var msg = mHandler.obtainMessage()
    msg.obj = result.body()?.data
    mHandler.sendMessage(msg)
}

val mHandler = Handler(Looper.getMainLooper()) {

    val result = it.obj as UserInfo
    mTextView.text = result.toString() // 更新控件 UI
    mProgressDialog?.dismiss() // 隐藏加载框

    false
 }
```

* 传统的方式需要启一个线程
* 执行请求，得到结果之后使用Handler将结果发送给主线程
* 主线程更新UI

##### 2.2下面是使用协程的方式

```kotlin
mMainScope.launch {
    Log.d("lpf", Thread.currentThread().name)
    mProgressDialog = ProgressDialog(this@Coroutines2Activity)
    mProgressDialog.setTitle("请求服务器中...")
    mProgressDialog.show()
    mTextView.setText("请求服务器中...")

    //可以完美解决网络嵌套的问题
    var result = RetrofitClient.instance.initRetrofit(WanAndroidAPI::class.java)
        .loginCoroutine("lpf666", "123456")

    mTextView.text = result.data.toString()// 更新控件 UI
    mProgressDialog?.dismiss() // 隐藏加载框
}
```

协程的方式这样就实现了，不需要创建子线程，不需要发消息，代码非常简洁，网络层只需要加上suspen关键字就可以。

协程可以解决异步回调的问题、协程同样可以解决多层网络嵌套的问题。

使用过程中注意的点：

* GlobalScope：全局作用域，默认是子线程，一般可以测试使用，不要直接使用否则可能会发生内存泄漏。
* MainScope：这个是局部作用域，可以直接在activity使用这个Scope，注意需要在OnDestroy进行释放
* viewModelScope：这个也是局部作用域，默认主线程，是ViewModel中自带的一个不需要创建直接使用就行。

协程可以轻松进行线程切换

```kotlin
mMainScope.launch(Dispatchers.Main) {
}
```

设置Dispatchers.IO()就是子线程了。



#### 3.协程的挂起与恢复

执行协程部分的代码逻辑，会自动挂起到IO线程，执行完成会自动恢复Main线程。

每一次从主线程到IO线程，都是一次协程挂起（suspend）

每一次从IO恢复到主线程，都是一次协程的（resume）

挂起，只是将程序执行流程转移到其他线程，并未阻塞主线程。

 ```kotlin
 var result = RetrofitClient.instance.initRetrofit(WanAndroidAPI::class.java)
         .loginCoroutine("lpf666", "123456")
 ```

以这行代码为例=号的左边的部分就是主线程，右边的部分就是子线程，这是协程自动挂起的。

使用协程会看到小图标：

* 可以挂起执行异步操作
* 可以恢复执行主线程

协程还需要遵守颜色规则，这一点很关键





#### 4.协程背后的状态机原理

将执行协程的代码反编译之后，可以看到一个Continuation

```kotlin
@kotlin.SinceKotlin public interface Continuation<in T> {
    public abstract val context: kotlin.coroutines.CoroutineContext

    public abstract fun resumeWith(result: kotlin.Result<T>): kotlin.Unit
}
```

其实本质上也是callback的方式，只是换成了Continuation，继续连续，通过resumeWith来恢复。

Continuation是继续执行后后面的意思，继续做接下来没做完的事情。

其实就是状态机的状态label在记录状态，label=0之后就完成了所有的逻辑。



#### 5.协程+MVVM+Jecpack项目架构

##### 5.1Google Jecpack+MVVM架构设计

![image-20220518224600055](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220518224600055.png)

Activity、Fragment通过ViewModel解决数据跟View耦合的问题，将逻辑拆分到几个不同的model中去，只需要关心数据的变化就可以，采用数据驱动UI的方式，Activity需要再处理View的刷新逻辑，只需要绑定ViewModel就可以。

ViewModel获取数据，如果是本地数据直接获取Room数据库的数据。

如果是网络数据，直接通过retrofit请求网络。



##### 5.2.协程+Retrofit+MVVM+DataBinding

![image-20220518224303769](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220518224303769.png)

* 通过Retrofit对网络接口封装并添加协程挂起suspend关键字，完成所有API接口。
* 通过Repository仓库，对数据进行统一管理，暴露给ViewModle层。
* ViewModel获取到数据之后通过DataBinding直接通过数据变化完成了页面的刷新。
* 协程的作用就是不需要接口进行回调了。



