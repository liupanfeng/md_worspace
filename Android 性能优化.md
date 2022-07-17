### Android 性能优化

Android性能是跟多个方面相关的，需要不断的优化才能把程序调优。

**代码层面的优化**

代码层面优化是最基本的能力，每个研发人员都应该努力提高自己的编码能力，从而写出高效的代码。

编写代码要遵循两个准则

不要做冗余的工作

避免次数过多的内存操作



**数据结构的选择**

正确的选择数据结构很重要，java常见的数据结构：ArrayList、LinkedList、HashMap、HashSet等，首先需要了解这些数据结构的区别和联系才能选择出合适的数据结构。

在Android研发过程中，可以使用SparseArray来代替HashMap，SpareArray是Android平台特有的稀疏数组实现的。它是Integer->Object的一个映射，在特定场合可用于代替HashMap提高性能，核心实现是二分法查找算法。

HashMap<Integer,Boolean>    --->SparseBooleanArray

HashMap<Integer,Integer>    --->SparseIntArray

HashMap<Integer,Long>    --->SparseLongArray

HashMap<Integer,String>    --->SparseArray<String>

SparseArray 不是线程安全的

由于要进行二分查找，所以添加的数据的key会进行大小顺序插入

SparseArray对删除做了优化，没有立即删除而是通过设置标识的方式，后边会尝试重用。

项目中遇到这些情况，都可以使用后边的数据结构进行替换，可以提升性能。



**Handler的正确使用**

```java
public class MainActivity extends AppCompatActivity {
    
    private final Handler mHandler=new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };
    ……
}
```

如果通过非静态内部类的方式使用Handler，由于非静态匿名内部类会持有外部类的引用，Handler 和MessageQueue与Message一起工作，当Message没处理完的情况下，activity是不会被释放的，所以会造成内存泄漏，会造成性能损耗。

```java
public class MainActivity extends AppCompatActivity {

    private static class MainHander extends Handler{

        private WeakReference<MainActivity> weakReference;

        public MainHander(MainActivity activity){
            this.weakReference=new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (weakReference.get()!=null){
                //……
            }
        }
    }

    private MainHander mMainHander=new MainHander(this)
        ……
}
                                                  
```

通过这个方式改写便不会存在内存泄漏的风险。



**正确的使用Context**

众所周知，context可以做很多事情：

```java
mContext.getResources();

mContext.startActivity(new Intent(this,MainActivity.class));

mContext.getSystemService(Context.TELEPHONY_IMS_SERVICE);

mContext.getCacheDir();

mContext.getExternalCacheDir();
```

根据Context的类别的不同，可以将context分为如下几个类别：

**Activity/Service**:这两个类都是ContextWrapper的子类，通过getBaseContext()方法可以获取到Context的实例，他们的context是独立的，不可复用。

**BroadcastReceiver**:跟activity不同，本身并不是context，回调方法onReceiver方法中系统会传入一个context，这个context是经过处理的，不能用来调用registerReceiver以及bindServer

**ContentProvider**：同样的本身也不是一个context的子类，创建的时候系统会传入一个context，可以通过getContext()方法得到，如果调用者跟contentPrivider处于同一个进程，那么得到的就是全局唯一的context，如果不在同一个进城，将持有自身所在进城的context。

**Application**：android中的默认单例类，在activity和service中可以通过context.getApplicationContext得到全局唯一的context。



如果应用程序中一个单例类需要注入context，这个如果注入Activity或者Service中的context会有内存泄漏的风向，正确的做法是注入Application中的context。

java中的四种引用

**强引用**

java中应用最广泛的一种，也是默认的引用类型。如果一个对象具有强引用，那么垃圾回收器是不会进行回收操作的，当内存不足的时候，java虚拟机会抛出OutOfMemoryError的一场，这时应用将会终止。

**软引用**

一个对象如果只有软引用，那么当内存充足时，垃圾回收器不会对它进行回收操作，只有当内存不足的时候，这个对象才会被回收。软引用可以用来实现内存敏感的高速缓存，如果配合引用队列（ReferenceQueue）使用，当软引用指向的对象垃圾回收器回收的后，java虚拟机将会把这个软引用加入到与之关联的引用队列当中。

**弱引用**

有的时候我们可能分不清楚软引用和弱引用的区别，弱引用时一种比软引用更弱的引用关系，只有弱引用指向对象的声明周期会更短，当垃圾回收期扫描到只有弱引用引用的对象的时候，不管内存是否充足，都将会被回收。弱引用可以和一个引用队列配合使用，当弱引用指向的对象被垃圾回收期后，java虚拟机会将这个弱引用的对象加入到这个弱引用队列当中。

**虚引用**

和软引用和弱引用不同的是，虚引用并不会对所引用对象的声明周期产生任何影响，也就是对象还是按照它原来的方式被垃圾回收期回收，虚引用本质上是一个标记作用，主要用来跟踪对象被垃圾回收器回收活动，虚引用必须和引用队列配合使用，当对象被垃圾回收期回收时，当这个对象存在虚引用，java虚拟机会将这个虚引用加入到与之关联的引用队列里边。



**避免创建非必要的对象**

对象的创建需要内存分配，对象的销毁需要垃圾回收，这些都会在一定程度上影响到应用的性能。因为最好是重用对象而不是直接创建，尤其需要避免在循环中重复创建相同的对象。



**对常量使用static final 修饰**

对于基本的数据类型和string类型的常量，建议使用static final修饰，因为final类型的常量会进入静态dex文件的初始化部分，这时对基本数类型和String类型常量的调用不会涉及类的初始化，而是直接使用字面量。



**避免内部的get/set方法**

使用get/set获取变量主要为了对外屏蔽具体的变量定义，从而达到更好的封装性。如果是在类内部还使用get/set来操作变量的话，会降低访问的速度，当没有使用JIT编译器的时候，直接的访问速度是使用 get/set访问的速度的3倍，如果是JIT编译器，速度会降低7倍。如果应用作用使用了ProGuard的话，那么ProGuard进行内联操作，从而达到直接访问的效果。



**代码的重构**

代码的重构是一项长期的持之以恒的工作，需要依靠团队每一个成员来维护库的高质量，如何有效的代码重构，除了需要对你所在项目有较深入的理解之外，你还需要一定的方法指导。可以学习《重构-改善既有代码的涉及》和《代码整洁之道》这两本书。



**图片的优化**

Android中能够使用的图片编码格式有三种：JPEG、PNG、WebP，图片格式可以通过查看Bitmap类的CompressFormat枚举值来确定。



**JPEG格式**

JPEG是一种广泛使用的有损压缩图像标准格式，它不支持透明和多帧动画，一般影视类作品最终会以JEEG格式展示。通过压缩比，可以调整图片的大小。



**PNG格式**

PNG是一种无损压缩图片格式，它支持完整的透明通道，从图片处理领域讲，JPEG只有RGB三个通道，而PNG有ARGB四个通道，由于是无损压缩，因此PNG格式的图片一般占用的格式比较大，会无形中增加app的大小，在做app瘦身时一般都会对PNG图片进行处理以减少其占用的体积。



**GIF格式**

支持多帧动画，各种动态的表情都是使用的这个格式。



**WebP格式**

webp是一个比较新的格式，它支持有损和无损压缩、支持完成的透明通道、也支持多帧动画，是一种比较理想的图片格式。在既要保证图片的质量又要限制图片大小的需求下，WebP因该是首选。



图片的压缩

目前项目里边，大多数的app使用的图片几乎都是png格式的资源，都会面临对PNG格式图片瘦身的要求，可以通过几个工具对PNG格式图片进行压缩

无损压缩ImageOptim

这个压缩工具，通过优化PNG压缩参数，移除冗余的数据以及非必须的颜色配置文件等方式，在不牺牲图片质量的前提下，既减小了PNG图片占用的空间又提高加载的速度。



有损压缩ImageAlpha

这个压缩工具，图片的大小会得到极大的降低，当然图片质量同时会受到影响，经过压缩的图片需要经过UI设计师检视才能上线，否则会影响整个app的视觉效果。

TinyPNG

也是一个有损的压缩工具，是WEB工具下，压缩之后的图片同样需要UI设计师检视。



PNG/JPEG 转Webp

根据Google测试，无损压缩后的WebP格式的图片比PNG格式文件小了45%，PNG格式的即便使用ImageOptim压缩之后，webP依然可以减少28%的文件大小。文件转换工具可以选择智图和iSparta等



尽量使用NinePatch格式的PNBG图

sdk/tools/draw9patch，点击几个启动制作工具，AndroidStudio也集成了转换工具。



图片缓存

这一快可以直接使用开源的图片框架进行处理，Glide，Fresocol等，对缓存都做了很好的缓存处理。



**电量优化**

手机的耗电是衡量一款App的重要指标之一，特别对于地图、导航、运动类的App，手机耗电有可能成为用户选择产品的一个重要因素。



BroadcastReceiver

为了减少耗电，我们的应用程序应该尽量减少无用的广播的执行。例如当应用退出到后台，一切的界面刷新都是没有意义的而且浪费内存和电量的。一个经典的场景，比如应用中存在一个监听网络状态变化的广播并执行一些动作，例如弹出Toast提示网络环境切换，当应用退出到后台的时候，这个广播的监听是没有意义的，就当应用退到后台就需要在onPause之后取消广播监听操作，根据具体的业务需求来选择当应用处于后台的时候是否需要监听广播。



数据传输

Android中的数据传输方式有很多种，常见的有：

蓝牙传输

蓝牙传输

移动网络传输

无论哪一种传输方式，为了更好的延长电池的使用时间，我们使用过程中需要重点关注两件事情：

后台数据传输的管理：根据业务需求，严格限制应用位于后台时是否禁用某些数据传输，尽量避免无效数据的传输。

数据传输的频度问题：通过经验值或者数据统计的方法确定好数据传输的频度，避免冗余重复的数据传输，数据传输过程中要压缩数据大小，合并网络请求，避免轮询等。



位置服务

位置服务已经成为应用必备的功能之一，特别时地图类、电商类和社交类应用，定位服务更是关键功能之一，能否有限地使用位置服务，是应用耗电的一个关键，Android中常见的位置服务有两种：GPS定位和网络定位。GPS定位服务需要ACCESS_FINE_LOCATION权限，网络定位服务需要ACCESS_COARSE_LOCATION或者ACCESS_FINE_LOCATION权限。代码中使用位置服务时，通常需要关注一下几个方面：

有没有及时注销位置监听器：和广播监听器一样，位置监听器也需要及时的注销，因为长时间的监听位置会耗费大量的电量，通常可以选择在页面的onPause中进行注销操作，更很用且全局有效的做法时禁用位置监听器。

位置更新监听频率的设定：根据具体的业务需求设置一个合适的更新频率值，通常需要在定位精度和耗电之间综合考虑。Android SDK提供了requestLocationUpdates方法进行设置，这个方法有两个主要的参数：

minTime：用来指定位置更新通知的最小时间间隔，单位时号秒。

minDistance：用来指定位置更新通知的最小距离，单位是米。

多种位置服务的选择：Android系统为我们提供了三种定位服务：

GPS定位：通过接受全局定位系统的卫星提供的经纬度坐标信息实现位置服务，精度最高。

网络定位：通常移动通信的基站信号差异来计算出手机所在的位置。

被动服务：最省电的定位服务，如果应用使用的被动定位服务，说明它想指导位置新信息但又不想主动获取，也就是这个应用会等待手机中其他应用、服务或者洗头工组件发出定位请求，并和这些组件的监听器一起接受位置更新。

应用中需要综合考虑应用的具体需求在不同时机采用不同的定位服务，实际开发中通常会选择四三方的SDK：百度、高德、腾讯服务。



**AlarmManager**

AlarmManager 是Android SDK 提供的一个唤醒API，它是系统级别的服务，可以在特定的时刻广播一个指定的Intent，这个PendingIntent可以用来启动Activity、Service或BroadcastReceiver。后台上传统计信息，可以通过一个AlarmManager来定时检查是否满足上传记录。AlarmManager提供了三个常用的方法。

set：设置一次性的闹钟操作。

setRepeating：设置重复性的闹钟操作。

setInexactRepeating：也是设置重复性的闹钟操作，只不过两个相连的闹钟执行的间隔时间是固定的。

AlarmManager的唤醒操作也是比较耗电的，通常情况下需要保证两次唤醒操作的时间间隔不要太短，在不需要使用唤醒功能的情况下尽早取消AlarmManager，否则会一直处于耗电状态。



**WakeLock**

wakeLock是为了保持设备处于唤醒状态的API，因为在某些情况下，即使用户长时间不交互，仍然需要阻止设备进入休眠状态，从而保证良好的用户体验，例如用户手机观看电影的时候，需要保障屏幕就保持开启状态。WakeLock的锁哟与很多种，不同的锁的类型和屏幕和键盘的影响不相同，具体情况如下。

PARTIAL_WAKE_LOCK:保持CPU正常运转，但屏幕和键盘灯可能是关闭的。

SCREEN_DIM_WAKE_LOCK:保持CPU正常运转，允许屏幕点亮但可能是置灰的，键盘灯可能是关闭的。

SCREEN_BRIGHT_WAKE_LOCK:保持CPU正常运转，允许屏幕高亮显示，键盘可能是关闭的。

FULL_WAKE_LOCK:保持CPU正常运转，屏幕高亮显示，键盘灯也保持亮度。

ACQUIRE_CAUSES_WAKEUP:当WakeLock被释放后，继续保持屏幕和键盘灯开启一定时间。



用WakeLock时，需要切换及时释放锁，有的在使用完毕WakeLock之后会忘记释放它，从而导致屏幕一直显示很长时间，快速的耗费手机电量，通常情况下，我们要尽早地释放WakeLock。例如在视频播放时获取WakeLock保持屏幕常量，在暂停播放的时候应该及时释放而不是等到停止播放才释放，在应用恢复播放时再次获取WakeLock即可。



**布局优化**

在进行Android应用的界面编写时，如果创建的布局层次结构比较复杂，View树嵌套的层比较深，那么将会使得页面的展现的时间比较长，导致应用越来越慢。

Android布局优化是实现应用响应灵敏的基础。遵循一些通用的编码规则有利于实现这个目标。

**include标签共享布局**

**ViewStub标签实现延迟加载**

**merge标签减少布局层次**

**尽量使用CompoundDrawable**

使用





美白的这个问题，现在的是参数设置为0，，之前关闭的时候是直接删除这个特技，我最新的包，设置的就是0了，，我再查一遍参数，，如果跟3.1.0不一样就只能暂时回复给你了























































































