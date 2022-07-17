### Kotlin-Service技术点



**1.Service的定义**

Service是Android中实现程序后台运行的解决方案，它非常适合执行那些不需要和用户交互还要长期运行的任务。



**2.Service运行所在环境**

Service不不是运行在独立的进程，而是依赖于创建Service时所在应用进程，当所在的应用程序进程被杀掉时，所有依赖该进程的Service都会被杀死。



**3.Service执行代码逻辑是否会阻塞主线程**

Service并不会自动开启线程，所有代码逻辑默认运行在主线程，如果直接执行耗时的任务就会有阻塞主线程的风险，所以如果需要执行耗时操作需要创建子线程进行处理。



**4.创建MyService时Exported和Enabled参数**

Exported这个参数表示是否暴露给外部程序使用

Enabled这个参数表示服务是否启用这个服务



**5.启动与停止服务方法一**

```kotlin
//启动服务
val intent = Intent(this,MyService::class.java)
startService(intent)

//停止服务
 val intent = Intent(this,MyService::class.java)
 stopService(intent)
```

这样就可以启动Service，服务会执行onCreate方法和onStartCommand方法，需要首次进行初始化的内容放在onCreate方法里边执行。

服务内部也可以调用stopSelf方法来停止服务。

多次调用startService，onCreate方法只会执行一次，onStartCommand每次启动都会执行。





**6.启动与断开服务方法二**

```kotlin
 private val connection = object : ServiceConnection {

        override fun onServiceDisconnected(name: ComponentName?) {
            Log.d("MainActivity","onServiceDisconnected")
        }

        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            Log.d("MainActivity","onServiceConnected")
        }

    }
	
//bind service
var intent=Intent(this,MyService::class.java)
bindService(intent,connection, Context.BIND_AUTO_CREATE)

//unbind Service
unbindService(connection)
```

通过bindService启动服务，需要先定义一个ServiceConnection，这个connection需要重写onServiceConnected这个方法服务连接的时候会回调，另外一个是onServiceDisconnected,这个方法是断开服务的时候回调。

通过bindService启动服务，Service会执行onCreate、onBind方法然后执行onServiceConnected方法。

如果通过两个方式启动了服务，需要调用stopService和unBinderService才会彻底的停掉服务的。





**7.Activity和Service进行通信**

如果需要进行通信，比如在activity中需要得到服务的执行过程就需要使用bindService的方式启动服务

```kotlin
class MyService : Service() {
	
    //定义并初始化binder成员变量
    private val mBinder = DownloadBinder()
	
    //在onBinder中返回会binder对象
    override fun onBind(intent: Intent): IBinder {
        Log.d("MyService","onBind")
       return mBinder
    }

    override fun onCreate() {
        super.onCreate()
        Log.d("MyService","onCreate")
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        Log.d("MyService","onStartCommand")
        return super.onStartCommand(intent, flags, startId)
        //执行到某个条件，可以调用这个方法自动停止Service
		//停止服务的方法
//       stopSelf()
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.d("MyService","onDestroy")
    }
	
    //定义binder内部类
    class DownloadBinder:Binder(){
        private var i:Int=0
        fun startDownload(){
            Log.d("MyService","startDownload")
            i++
        }

        fun getProgress():Int{
            Log.d("MyService","getProgress = "+i)
            return i
        }
    }


}
```



接下来就是在activity通过binder来得到需要用到的内容

```kotlin
private val connection = object : ServiceConnection {

    override fun onServiceDisconnected(name: ComponentName?) {
        Log.d("MyService","onServiceDisconnected")
    }

    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        Log.d("MyService","onServiceConnected")
        downloadBinder=service as MyService.DownloadBinder
        downloadBinder.startDownload()
        downloadBinder.getProgress()
    }

}

fun onBind(view:View){
    var intent=Intent(this,MyService::class.java)
    bindService(intent,connection, Context.BIND_AUTO_CREATE)

}
```

在onServiceConnected中得到binder对象，通过这个对象就可以获取到服务的执行进度等需要的信息。



**8.前台服务**

从Android8.0系统开始，只有当应用保持在前台可见状态的情况下，Service才能保证稳定运行，一旦应用程序进入后台之后，Service随时都可能被系统回收。如果希望Service一直保持运行状态，就需要考虑使用前台服务。前台服务和普通服务的区别是，前台服务一直有一个正在运行的图标在系统的状态栏显示，下拉状态栏可以清晰看到详细信息。

```kotlin
class MyService : Service() {

   ...
    
   override fun onCreate() {
        super.onCreate()
        Log.d("MyService","onCreate")

        val manager=getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        if (Build.VERSION.SDK_INT >=Build.VERSION_CODES.O){
            val channel=NotificationChannel("my_service","前台服务",NotificationManager.IMPORTANCE_DEFAULT)
            manager.createNotificationChannel(channel)
        }

        val intent=Intent(this,MainActivity::class.java)
        val pi=PendingIntent.getActivity(this,0,intent,0)
        val builder=NotificationCompat.Builder(this,"MyService")

        builder.setContentTitle("this is content title")
            .setContentInfo("this is content info")
            .setSmallIcon(R.mipmap.ic_launcher)
            .setLargeIcon(BitmapFactory.decodeResource(resources,R.mipmap.ic_launcher))
            .setContentIntent(pi)


        if (Build.VERSION.SDK_INT >=Build.VERSION_CODES.O) {
            builder.setChannelId("my_service")
        }
        val notification=builder.build()
        startForeground(1, notification)
    }

    ...
}
```

只需要在onCreate方法中添加相关的代码逻辑就可以，另外在Android9.0以上，启动前台服务需要申明权限才行

android.permission.FOREGROUND_SERVICE

做好上面这些之后，执行startService就可以启动前台服务了……















