### android-UI性能分析工具systrace

[TOC]

#### 1.systrace介绍

* **systrace**是一款UI性能数据采样和分析工具，它可以帮助开发者收集Android关键子系统（如**surfaceflinger**、**WindowManagerService**等Framework部分关键模块、服务，View系统等）的运行信息，从而帮助开发者更直观地分析系统瓶颈，改进性能。
* **Systrace**的功能包括跟踪系统的I/O操作、内核工作队列、CPU负载等在UI显示性能分析上提供很好的数据，特别是在动画播放不流畅、渲染卡等问题上。

* **Systrace**工具可以跟踪、收集、检查定时信息，可以很直观地查看CPU周期消耗的具体时间，显示每个线程和进程的跟踪信息，使用**不同颜色来突出问题的严重性**，并提供如何解决这些问题的建议。

#### 2.systrace使用

* systrace这个工具在android-sdk\platform-tools\systrace 存在这个路径下面

* systrace工具是一个python的可执行文件，所以需要使用到python的环境，本地使用的python版本2.7的版本，高版本可能有兼容问题。

* ```shell
  在命令函执行这个就可以手机systrace数据
  python systrace.py -a com.meishe.sdkdemo -b 16384 -t 10  gfx input view webview  sm freq idle sched  -o test_trace.html
  ```

  * -a ：执行应用程序包名
  * -b ：buffer 单位是kb 占用多少缓存 
  * -t ：时间，单位是毫秒
  * gfx、input、view、webview、sm、freq、idle、sched 表示手机信息的维度。在官网可以查看具体的介绍http://developer.android.com/intl/zh-cn/tools/help/systrace.html
  * -o 输出的文件名

* 生成trace文件之后，打开google浏览器，输入：chrome://tracing，选择Perfetto_UI 这个工具很强大，需要使用新版本的google浏览器才可以，如果打不开升级浏览器，直接输入这个https://ui.perfetto.dev/是一样的。

* Open with legacy UI 这个是老版本，直接选择生成的trace文件就可以查看。

* Open trace file ：这个是新版本，推荐使用这个。



#### 3.systrace 添加flag

##### 3.1收集应用启动的systrace数据

在Application的attachBaseContext方法添加ms_init flag 

```java
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    MultiDex.install(base);
    Trace.beginSection("ms_init");
}
```

在MainActivity onWindowFocusChanged方法结束

```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if (hasFocus){
        Trace.endSection();
    }
}
```

通过筛选ms_init标记，就可以得到启动过程的systrace数据了。



##### 3.2在Fragment的onViewCreate方法的前后添加flag，可以方便分析fragment的view加载耗时。

##### 3.3如果RecyclerView的滑动卡顿，在onBindViewHolder方便前后添加编辑，可以方便分析item的绘制耗时。

打上标记之后，可以通过FlowEvent来找到点来计算时间。



#### 4.分析systrace报告

* 快捷键W-放大、S-缩小、A-左移、D-右移
* Alerts一栏标记了性能有问题的点，单击该点可以查看详细信息，在右边侧边栏还有一个Alerts框，单击可以查看每个类型的Alerts的数量，单击某一个Alert可以看到问题的详细描述。
* 每个应用都有一行专门显示frame，每一帧就显示为一个绿色的圆圈。当显示为黄色或者红色时，它的渲染时间超过了16.6ms（即达不到60fps的水准）。使用W键放大，看看这一帧的渲染过程中系统到底做了什么，同时它会将任何它认为性能有问题的东西都高亮警告，并提示要怎么优化
* Choreographer 会有一帧的回调onFrame    有三次的回调：traggle double   回调的时间长还是加载的问题。这是一个性能瓶颈点的查找点。

**systrace的时候 Frame、alert、时间分片、具体app的Cpu和Gpu**

线上的app 反射来操作 ，反射android.os.trace  执行setAppTracingAllow 设置这个方法设置为true



#### 5.UI绘制分析

使用systrace工具很简单，但是看懂里边的内容就需要了解view的绘制过程

![image-20220509113556881](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220509113556881.png)

* Choreographer 会发送垂直同步信号 onFrame，两个信号之前是16.6毫秒。
* 两次垂直信号之间需要处理的内容包含：
  * UI线程的输入、动画、测量、摆放、绘制、sync同步给GPU
  * RenderThread绘制线程 sync同步、execute执行一些操作、获取缓冲区、交换缓冲区
  * SurfaceFlinger Graphics 交换缓冲区的数据，进行合成展现到设备中去



