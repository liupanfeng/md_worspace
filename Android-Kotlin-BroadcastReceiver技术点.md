### Android-Kotlin-Broadcast技术点

Android中为了便于系统级别的消息通知，引入了广播机制。如果想接收到广播就必须要注册广播接收者-BroadcastReceiver

**1.广播的分类**

* 标准广播：是一种完全异步的广播，发出后所有的接收者会几乎同时接收到，没有向后顺序可言，这种广播效率比较高。
* 有序广播：是一种同步广播，同一时刻只有一个接收者接收到这个广播，并且有优先级的概念，优先级高的接收者先接收到这个广播。接收到这个广播的接收者可以终止这个广播。



**2.注册广播**

* 动态注册：在代码中注册的接收者，需要在相对应的声明周期里边进行反注册。

  ```Kotlin
  
    override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContentView(R.layout.activity_broadcast)
          register()
      }
  
  /**
   * 动态注册广播
   */
  private fun register() {
      val intentFilter=IntentFilter()
      intentFilter.addAction("android.intent.action.TIME_TICK")
      receiver=TimeChangeReceiver()
      registerReceiver(receiver,intentFilter)
  }
  
  
  inner class TimeChangeReceiver :BroadcastReceiver(){
      override fun onReceive(context: Context?, intent: Intent?) {
           showToast("time change")
      }
  
  }
  
    override fun onDestroy() {
          super.onDestroy()
          //取消注册广播
          unregisterReceiver(receiver)
      }
  
  ```

* 静态注册：在AndroidManifest清单文件注册的广播接收者，应用未启动也能收到广播，并且在广播中执行相关的代码逻辑。

  ```kotlin
  class BootCompleteReceiver : BroadcastReceiver() {
  
      override fun onReceive(context: Context, intent: Intent) {
          // This method is called when the BroadcastReceiver is receiving an Intent broadcast.
          showToast("reboot already")
      }
  }
  
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <application
              ...
              android:theme="@style/AppTheme">
  
          <receiver
                  android:name=".broadcast.BootCompleteReceiver"
                  android:enabled="true"
                  android:exported="true">
  		 <intent-filter>
                  <action android:name="android.intent.action.BOOT_COMPLETED"/>
              </intent-filter>
          </receiver>
  
       ...
      </application>
  
  ```

 这样就注册了一个静态的广播，能接听到开机的广播并弹出Tooast。exported=“true” 表示允许接收除了本程序以外的广播，enabled=“true”表示开启这个广播接收者。

Android系统为了保护用户设备的安全和隐私：如果程序需要进行一些敏感的操作，必须声明权限，上面这个启动广播就需要声明权限否则会崩溃。

```kotlin
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

重启手机，便收到了开机广播。



**3.隐式广播**

隐式广播是指那些没有具体指定发送给那个应用程序的广播。**Android系统8.0之后所有的隐式广播都不允许使用静态注册的方式注册来接收了**。少数特殊的系统广播目前还能使用，比如：RECEIVE_BOOT_COMPLETED。我们发送的自定义广播也是隐式广播，如果希望能被自己的应用程序接收到，**需要显式指定包名才可以。**



**4.发送广播**

静态注册自定义的广播接收者，并指定Action

```Kotlin
class MyReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        // This method is called when the BroadcastReceiver is receiving an Intent broadcast.
       showToast("receive my Broadcast" )
    }
}

 //AndroidManifest添加
     <receiver
                android:name=".broadcast.MyReceiver"
                android:enabled="true"
                android:exported="true">
            <intent-filter>
                <action android:name="com.test.kotlin_test.MY_RECEIVER"/>
            </intent-filter>
        </receiver>
```



发送标准广播，注意需要设置应用的包名，intent.setPackage("com.test.kotlin_test"）指明这个广播是发送给那个应用程序的，这样这个广播就不是隐式广播了，否则静态注册接收不到。

```Kotlin
fun sendBroadcast(view: View){
    val intent=Intent()
    intent.action = "com.test.kotlin_test.MY_RECEIVER"
    intent.setPackage("com.test.kotlin_test")
    sendBroadcast(intent)
}
```





**5.发送有序广播**

```kotlin
fun sendOrBroadcast(view: View){
    val intent=Intent()
    intent.action = "com.test.kotlin_test.MY_RECEIVER"
    intent.setPackage("com.test.kotlin_test")
    sendOrderedBroadcast(intent,null)
}
```

发送有序广播只需要使用这个方法：sendOrderedBroadcast，第二个参数是跟权限相关的，可以暂时传null，接收者优先级高的先接收到有序广播，可以调用abortBroadcast()这个方法终止广播。其他的接收者就不能收到这个广播的。



```kotlin
<receiver
        android:name=".broadcast.OtherReceiver"
        android:enabled="true"
        android:exported="true">
    <intent-filter android:priority="100">
        <action android:name="com.test.kotlin_test.MY_RECEIVER"/>
    </intent-filter>
</receiver>
```

在intent-filter中可以配置priority 这个是广播接收者的优先级，默认是0，值越大优先级越高，取值范围是-1000，1000之间的数值。



**6.广播的应用，强制退出登录功能**

强制退出功能，可以使用广播的功能来实现，思路就是当需要强制退出的时候发送广播，接收到广播之后弹出弹窗，用户确认退出之后杀死所有的Activity并启动登录页面。



```kotlin
object ActivityController {
    private val activitys=ArrayList<Activity>()

    fun addActivity(activity: Activity){
        activitys.add(activity)
    }

    fun removeActivity(activity: Activity){
        activitys.remove(activity)
    }

    fun finishAll(){
        for (activity in activitys){
            if (!activity.isFinishing){
                activity.finish()
            }
        }
        activitys.clear()
    }
}
```

需要关闭所有的Activity，所以需要建立一个Activity的管理者，这个来管理Activity



```kotlin
open class BaseActivity :AppCompatActivity() {

    private lateinit var receiver:ForceOfflineReceiver

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ActivityController.addActivity(this)
    }

    override fun onDestroy() {
        super.onDestroy()
        ActivityController.removeActivity(this)
    }

    override fun onResume() {
        super.onResume()
        val intentFilter=IntentFilter()
        intentFilter.addAction("com.test.kotlin_test.FORCE_OFFLINE")
        receiver=ForceOfflineReceiver()
        registerReceiver(receiver,intentFilter)
    }


    override fun onPause() {
        super.onPause()
        unregisterReceiver(receiver)
    }

    inner class ForceOfflineReceiver :BroadcastReceiver(){
        override fun onReceive(context: Context, intent: Intent?) {
            AlertDialog.Builder(context).apply {
                setTitle("注意")
                setMessage("你正在操作强制退出登录")
                setCancelable(false)
                setPositiveButton("ok"){_,_->
                    ActivityController.finishAll() //销毁所有的Activity
                    val intent=Intent(context,LoginActivity::class.java)
                    context.startActivity(intent)
                }
                show()
            }
        }

    }
```

定义一个baseActivity，让其他所有的Activity都继承这个，方便管理其他的页面，同时注意广播接收者的注册和反注册分别是在onResume和onPause声明周期里边，这样可以及时的注册和反注册广播，接收到广播之后弹出弹窗，用户点击确定后首先关闭所有Activity然后启动登录页面。

这样强制退出的功能就实现了。

