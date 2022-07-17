### Android性能优化一绘制优化

Android应用启动慢，使用时经常卡顿，是非常影响用户体验的，应该尽量避免出现。



**1.卡顿的分类**

按照场景分可以分为：

* UI绘制
  * 绘制
  * 刷新
* 应用启动
  * 安装启动
  * 冷启动
  * 热启动
* 页面跳转
  * 页面间切换
  * 前后台切换
* 事件响应
  * 按键
  * 系统事件
  * 滑动



**2.卡顿的原因**

这4种卡顿场景的根本原因可以分成两大类：

**界面绘制**：主要原因是绘制的**层级深**、**页面复杂**、**刷新不合理**，由于这些原因导致卡顿的场景更多出现在UI和启动后的初始界面以及跳转到页面的绘制上。

**数据处理**：导致这种卡顿场景的原因是数据处理量太大，一般分为三种情况，一是数据处理在UI线程（这种应该避免），二是数据处理占用CPU高，导致主线程拿不到时间片，三是内存增加导致GC频繁，从而引起卡顿。



**3.Android系统的显示原理**

整个显示系统很复杂，对于性能优化了解整体流程就可以。

Android的显示过程可以简单概括为：Android应用程序把经过测量、布局、绘制后的surface缓存数据，通过SurfaceFlinger把数据渲染到显示屏幕上，通过Android的刷新机制来刷新数据。也就是说**应用层负责绘制**，**系统层负责渲染**，通过进程间通信把应用层需要绘制的数据传递到系统层服务，系统层服务通过刷新机制把数据更新到屏幕。

Android的图形显示系统采用的是Client/Server架构，SurfaceFlinger是一个c++文件（Server端），源码路径：frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp，感兴趣可以去深入分析研究。

**Client端代码**分为两部分，一部分是由Java提供给应用使用的API，另一部分则是由C++写成的底层具体实现。



**4.绘制的原理**

绘制任务是由应用发起的，最终通过**系统层绘制到硬件屏幕上**，应用进程绘制好后，通过**跨进程通信机制**把需要显示的数据传到系统层，由**系统层中的SurfaceFlinger服务绘制到屏幕上**。

在Android的每个View绘制中有三个核心步骤：通过Measure和Layout来确定当前需要绘制的View所在的大小和位置，通过绘制（Draw）到surface。

在Android系统中整体的绘图源码是在ViewRootImp类（**源码位置：frameworks/base/core/java/android/view/ViewRootImpl.java**）的**performTraversals()**方法，通过这个方法可以看出Measure和Layout都是递归来获取View的大小和位置，并且以深度作为优先级。可以看出，**层级越深，元素越多，耗时也就越长**。



**（1）应用层：**

**Measure**：用深度优先原则递归得到所有视图（View）的宽、高；获取当前View的正确宽度childWidthMeasureSpec和高度childHeightMeasureSpec之后，可以调用它的成员函数Measure来设置它的大小。如果当前正在测量的子视图child是一个视图容器，那么它又会重复执行操作，直到它的所有子孙视图的大小都测量完成为止。

**Layout**：用深度优先原则递归得到所有视图（View）的位置；当一个子View在应用程序窗口左上角的位置确定之后，再结合它在前面测量过程中确定的宽度和高度，就可以完全确定它在应用程序窗口中的布局。

**Draw**：目前Android支持了两种绘制方式：软件绘制（CPU）和硬件加速（GPU），其中硬件加速在Android 3.0开始已经全面支持，，硬件加速在UI的显示和绘制的效率远远高于CPU绘制，但硬件加速也存在明显的缺点：

* 耗电：GPU的功耗比CPU高

* 兼容问题：某些接口和函数不支持硬件加速。

* 内存大：使用OpenGL的接口至少需要8MB内存。

  

**（2）系统层**

真正把需要显示的数据渲染到屏幕上，是通过系统级进程中的SurfaceFlinger服务来实现的。

SurfaceFlinger主要负责的任务：

* 响应客户端事件，创建Layer与客户端的Surface建立连接。
* 接收客户端数据及属性，修改Layer属性，如尺寸、颜色、透明度等。
* 将创建的Layer内容刷新到屏幕上。
* 维持Layer的序列，并对Layer最终输出做出裁剪计算。。

既然两个不同进程，那么肯定需要一个跨进程的通信机制来实现数据传输，在Android的显示系统，使用了Android的匿名共享内存：SharedClient，每一个应用和SurfaceFlinger之间都会创建一个SharedClient，在每个SharedClient中，最多可以创建31个SharedBufferStack，每个Surface都对应一个SharedBufferStack，也就是一个window。

![image-20220403220135904](E:\md_workspace\绘制优化\surfaceFlinger工作原理.png)

**一个SharedClient对应一个Android应用程序**，而一个Android应用程序可能包含多个窗口，即Surface。也就是说SharedClient包含的是SharedBufferStack的集合。因为最多可以创建31个SharedBufferStack，也就是说一个Android应用程序最多可以包含31个窗口，同时每个SharedBufferStack中又包含了两个（低于4.1版本）或者三个（4.1及以上版本）缓冲区，即显示刷新机制中会提到的双缓冲和三重缓冲技术。



总结起来显示整体流程分为三个模块：1.应用层绘制到缓存区，2.SurfaceFlinger把缓存区数据渲染到屏幕，3.由于是两个不同的进程，所以使用Android的匿名共享内存SharedClient缓存需要显示的数据来达到目的。



cpu和gpu是如何系统工作的呢？

绘制过程首先是CPU准备数据，通过Driver层把数据交给CPU渲染，其中**CPU主要负责Measure、Layout、Record、Execute的数据计算工作**，**GPU负责Rasterization（栅格化）、渲染**。由于图形API不允许CPU直接与GPU通信，而是通过中间的一个图形驱动层（GraphicsDriver）来连接这两部分。图形驱动维护了一个队列，CPU把display list添加到队列中，GPU从这个队列取出数据进行绘制，最终才在显示屏上显示出来



**5.FPS帧率**

到底绘制一个单元多长时间才是合理的，需要FPS。FPS（Frames PerSecond）表示每秒传递的帧数。在理想情况下，60 FPS就感觉不到卡，这意味着每个绘制时长应该在16ms以内。



Android系统很有可能无法及时完成那些复杂的界面渲染操作。**Android系统每隔16ms发出VSYNC信号**，触发对UI进行渲染，如果每次渲染都成功，这样就能够达到流畅的画面所需的60FPS。即为了实现60FPS，就意味着程序的大多数绘制操作都必须在16ms内完成。



如果某个操作花费的时间是24ms，系统在得到VSYNC信号时就无法进行正常渲染，这样就发生了丢帧现象。那么用户在32ms内看到的会是同一帧画面。有很多原因可以导致CPU或者GPU负载过重从而出现丢帧现象：可能是**Layout太过复杂，无法在16ms内完成渲染**；可能是**UI上有层叠太多的绘制单元**；还有可能是**动画执行的次数过多。**最终的数据是刷新机制通过系统去刷新数据，**刷新不及时也是引起卡顿的一个主要原因**。



**6.双缓冲、VSYNC、Choreographer解释**



**双缓冲**：显示内容的数据内存，为什么要用双缓冲，我们知道在Linux上通常使用Framebuffer来做显示输出，当用户进程更新Framebuffer中的数据后，显示驱动会把Framebuffer中每个像素点的值更新到屏幕，但这样会带来一个问题，如果上一帧的数据还没有显示完，Framebuffer中的数据又更新了，就会带来残影的问题，给用户直观的感觉就会有闪烁感，所以普遍采用了双缓冲技术。双缓冲意味着要使用两个缓冲区（在SharedBufferStack中），其中一个称为Front Buffer，另外一个称为Back Buffer。UI总是先在Back Buffer中绘制，然后再和Front Buffer交换，渲染到显示设备中。即只有当另一个buffer的数据准备好后，通过io_ctrl来通知显示设备切换Buffer。



**VSYNC**：只有当另一个buffer准备好后，才能通知刷新，这就需要CPU以主动查询的方式来保证数据是否准备好，因为这种机制效率很低，所以引入了VSYNC。VSYNC是VerticalSynchronization（垂直同步）的缩写，可以简单地把它认为是一种定时中断，一旦收到VSYNC中断，CPU就开始处理各帧数据。



**Choreographer**：收到VSYNC信号时，调用用户设置的回调函数。一共有以下三种类型的回调：

*  CALLBACK_INPUT：优先级最高，与输入事件有关。
* CALLBACK_ANIMATION：第二优先级，与动画有关。
*  CALLBACK_TRAVERSAL：最低优先级，与UI控件绘制有关。



**7.卡顿的根本原因**

* 绘制任务太重，绘制一帧内容耗时太长。
* 主线程太忙了，导致VSync信号来时还没有准备好数据导致丢帧。



**8.主线程应该负责什么才是合理的**

* UI生命周期控制
* 系统事件处理
* 消息处理
* 界面绘制
* 界面刷新

除了这些以外，尽量避免将其他处理放到主线程中，特别是复杂的数据计算和网络请求。



未完待续……