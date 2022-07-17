### Android-布局优化

[TOC]

#### 1.减少层级

#### 1.1页面层级查看

通过使用布局view层级检测工具hierarchy viewer来查看页面的层级。这个工具在android-sdk/tools/文件夹中，默认无法连接真机，可以通过ViewServer来解决无法连接真机的问题，在需要查看的页面添加：

```java
//就可以解决无法连接真机的问题
@Override
    protected void onCreate(Bundle saveInstanceState) {
        super.onCreate(saveInstanceState);
        if (BuildConfig.DEBUG){
            ViewServer.get(this).addWindow(this);
        }
    }

 @Override
    protected void onResume() {
        super.onResume();

        if (BuildConfig.DEBUG){
            ViewServer.get(this).setFocusedWindow(this);
        }
    }

    @Override
    protected void onDestroy() {
        if (BuildConfig.DEBUG){
            ViewServer.get(this).removeWindow(this);
        }
    }

```

点击Evaluate Contrast可以View的颜色绿色、黄色、红色，出现黄色或者红色表示页面层级较深，需要优化。

也可以通过Lint进行代码扫描。

#### 1.2页面层级优化

层级越少，测试和绘制的时间就越短，通常减少层级有以下两个常用方案：

###### 1.2.1合理使用RelativeLayout和LinearLayout

* 层级一样优先使用LinearLayout，因为RelativeLayout默认进行两次measure操作。
* 用LinearLayout有时会使嵌套层级变多，应该使用RelativeLayout，使界面尽量扁平化。

###### 1.2.2合理使用Merge

在Android布局的源码中，如果是Merge标签，那么直接将其中的子元素添加到Merge标签Parent中，这样就保证了不会引入额外的层级。

Merge不是所有地方都可以任意使用，有以下几点要求：

* Merge只能用在布局XML文件的根元素。
* 使用merge来加载一个布局时，必须指定一个ViewGroup作为其父元素，并且要设置加载的attachToRoot参数为true。
* 不能在ViewStub中使用Merge标签。原因就是ViewStub的inf late方法中根本没有attachToRoot的设置。

但从Lint检查的配置上看，超过10层才会报警，实际上在开发时，随着产品设计的丰富和多样性，很容易超过10层，根据实际开发过程中超过15层就要重视并准备做优化， 20层就必须修改了

#### 2.提高显示速度ViewStub的使用

##### 2.1ViewStub介绍

ViewStub是一个轻量级的View，它是一个看不见的，并且不占布局位置，占用资源非常小的视图对象。可以为ViewStub指定一个布局，加载布局时，只有ViewStub会被初始化，然后当ViewStub被设置为可见时，或是调用了ViewStub.inflate()时，ViewStub所指向的布局会被加载和实例化，然后ViewStub的布局属性都会传给它指向的布局。这样，就可以使用ViewStub来设置是否显示某个布局。

##### 2.2ViewStub的加载方式：

ViewStub显示有两种方式：

* 调用：inflate()方法.
* 直接使用ViewStub. setVisibiltity（View.Visible）方法。

##### 2.3ViewStub应用场景

ViewStub的主要使用场景如下：

* 在程序运行期间，某个布局在加载后，就不会有变化，除非销毁该页面再重新加载。
* 想要控制显示与隐藏的是一个布局文件，而非某个View。



##### 2.4ViewStub使用注意事项 

使用ViewStub时需要注意以下几点：

* **ViewStub只能加载一次**，之后ViewStub对象会被置为空。换句话说，某个被ViewStub指定的布局被加载后，就不能再通过ViewStub来控制它了。所以它不适用于需要按需显示隐藏的情况。

* **ViewStub只能用来加载一个布局文件**，而不是某个具体的View，当然也可以把View写在某个布局文件中。如果想操作一个具体的View，还是使用visibility属性。
* VIewStub中不能嵌套Merge标签



#### 3.布局复用include

在开发过程中，如果一个相同的布局在很多页面（Activity或Fragment）会用到，如果给这些页面的布局文件都统一加上相同的布局代码，维护起来就很麻烦，可读性也差，一旦需要修改，很容易有漏掉的地方。

Android的布局复用可以通过<include>标签来实现，就像提取代码公用部分一样，在编写Android布局文件时，也可以将相同的部分提取出来，在使用时，用<include>添加进去。



#### 4.避免过度绘制

##### 4.1过度绘制介绍

**过度绘制（Overdraw）**是指在屏幕上的某个像素在同一帧的时间内被绘制了多次。在多层次重叠的UI结构（如带背景的TextView）中，如果不可见的UI也在做绘制的操作，就会导致某些像素区域被绘制了多次，从而浪费多余的CPU以及GPU资源。

##### 4.2过去绘制的原因

过度绘制的主要原因：

* XML布局->控件有重叠且都有设置背景
* View自绘-> View.OnDraw里面同一个区域被绘制多次

##### 4.3过度绘制的检测

可以通过手机设置中的开发者选项，打开Show GPU Overdraw选项

打开后可以根据不同的颜色观察UI上的Overdraw情况，蓝色、淡绿、淡红、深红代表4种不同程度的Overdraw情况，不同颜色的含义如下：

* 无色：没有过度绘制，每个像素绘制了1次。
* 蓝色：每个像素多绘制了1次。大片的蓝色还是可以接受的。如果整个窗口是蓝色的，可以尝试优化减少一次绘制。
* 绿色：每个像素多绘制了2次。
* 深红：每个像素多绘制了4次或者更多。严重影响性能，需要优化，避免深红色区域。

我们的目标是尽量减少红色Overdraw，看到更多的蓝色区域。

##### 4.4过度绘制的优化

###### 4.4.1布局文件过度绘制优化：较少页面层级，减少不必要的背景设置

使用Android自带的一些主题时，activity往往会被设置一个默认的背景，这个背景由DecorView持有。当自定义布局有一个全屏的背景时，比如设置了这个界面的全屏黑色背景，DecorView的背景此时对我们来说是无用的，但是它会产生一次Overdraw。可以通过下面的方式消除。

```java
@Override
protected void onCreate(Bundle saveInstanceState) {
    super.onCreate(saveInstanceState);
    getWindow().setBackgroundDrawable(null);
}
```



针对RecyclerView Item中的Avatar ImageView的设置，在getView的代码中，判断是否获取对应的Bitmap，获取Avatar的图像之后，把ImageView的Background设置为Transparent，只有当图像没有获取到时，才设置对应的Background占位图片，这样可以避免因为给Avatar设置背景图而导致的过度渲染。

###### 4.4.2自定义View优化过度绘制的优化

在自定义View的时候，某个区域可能会被绘制多次，造成过度绘制。

可以通过canvas.clipRect()方法指定绘制区域。

clipRect方法还可以帮助节约CPU与GPU资源，在clipRect区域之外的绘制指令都不会被执行，那些部分内容在矩形区域内的组件，仍然会得到绘制。

并且可以使用canvas.quickreject()来判断是否没和某个矩形相交，从而跳过那些非矩形区域内的绘制操作。

#### 5.子线程加载布局文件

使用AsyncLayoutInflater在子线程加载布局文件需要添加依赖：

```groovy
implementation 'androidx.asynclayoutinflater:asynclayoutinflater:1.0.0'
```

下面是通过子线程添加布局文件的例子：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    AsyncLayoutInflater(MainActivity@ this).inflate(
        R.layout.activity_main, null
    ) { view, resid, parent ->
        setContentView(view)
        val contentTextView: TextView =findViewById(R.id.content);
        contentTextView.text = "通过子线程加载layout布局文件"
    }
}
```



#### 6.使用x2c对布局进行加载，

setContentView加载xml布局文件，framework层主要是通过LayoutInflater.inflater方法来解析xml并创建View的，解析XML文件会涉及io读写，view的创建是通过反射来实现的，都是比较耗时的操作。

**可以借助x2c来加载布局文件，可以避免反射和io读取**

使用x2c加载布局文件需要导入一下依赖：

```groovy
annotationProcessor 'com.zhangyue.we:x2c-apt:1.1.2'
implementation 'com.zhangyue.we:x2c-lib:1.0.6'
```

使用x2c加载布局文件实例：

```java
@Xml(layouts = ["activity_main"])
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        X2C.setContentView(this, R.layout.activity_main)
        val contentTextView: TextView =findViewById(R.id.content);
            contentTextView.text = "通过X2C加载layout布局文件"
    }
}
```



x2c原理就是编译时注解：遍历注解，通过动态生成代码的方式，将反射的方式改成new的方式将View进行初始化操作。



x2c可能会存在兼容性问题，出现问题，需要做一些代码的优化。

