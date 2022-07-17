### 注入框架Hilt

**1.什么是Hilt**

Hilt是Android团队基于Dagger2，开发的一个专门面向Android的依赖注入框架，相比于Dagger2，Hilt具有一下的优势：

* 使用加单
* 提供了Android专属API



**2.Hilt在那些方面做了优化**

* 不再需要编写大量的Component代码
* Scope会与Component自动绑定



**3.Hilt 目前支持一下Android的类**

* Application  @InstallIn(ApplicationComponent.class)
* Activity         @InstallIn(ActivityComponent.class)
* Fragment     @InstallIn(FragmentComponent.class)
* View              @InstallIn(ViewComponent.class)
* Service          @InstallIn(ServiceComponent.class)
* BroadcastReceiver  @InstallIn(BroadcastReceiverComponentManager.class)



**4.使用Hilt需要引入的依赖项**

```groovy
//需要导入这个插件，因为hilt的实现原理包含字节码插桩 所以需要依赖这个
apply plugin: 'dagger.hilt.android.plugin' 

implementation "com.google.dagger:hilt-android:2.28-alpha" // hilt的支持
annotationProcessor "com.google.dagger:hilt-android-compiler:2.28-alpha" //Dagger的注解处理器APT

//在最外层还需要添加这个
classpath 'com.google.dagger:hilt-android-gradle-plugin:2.28-alpha' // 导入gradle-plugin 
```



**5.使用流程**

* 准备需要注入的对象

```java
public class CoilObject {
    @Override
    public int hashCode() {
        return super.hashCode();
    }
}
```

* 编写module

```java
@InstallIn(ActivityComponent.class)
@Module
public class CoilModule {

    @Provides
    public CoilObject provideCoilObject(){
        return new CoilObject();
    }
}
```

* 在Application中增加注解@HiltAndroidApp

```java
@HiltAndroidApp
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
    }

}
```

* 在Activty增加@AndroidEntryPoint 注解

* 进行通过@Inject进行对象注入

```java
@AndroidEntryPoint
public class MainActivity extends AppCompatActivity {

    @Inject
    CoilObject coilObject;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toast.makeText(this, "coilObject address is "+coilObject.hashCode(), Toast.LENGTH_SHORT).show();
    }
}
```

以上就是一个简答的使用过程



**6.实现单例的注入**

* 准备需要注入的对象

```java
public class CoilSingleton {
    @Override
    public int hashCode() {
        return super.hashCode();
    }
}
```



* 编写module，单例包含局部单例以及全局单例

```java
//@InstallIn(ActivityComponent.class)
@InstallIn(ApplicationComponent.class)
@Module
public class CoilSingletonModule {

    @Provides
//    @ActivityScoped    // 这样定义表示局部单例 需要跟ActivityComponent 搭配使用
    @Singleton          //这样定义表示全局单例  需要跟 ApplicationComponent 搭配使用
    public CoilSingleton provideCoilObject(){
        return new CoilSingleton();
    }


}
```



* 在activity进行注入

```java
@AndroidEntryPoint
public class MainActivity extends AppCompatActivity {

    @Inject
    CoilSingleton coilSingleton1;

    @Inject
    CoilSingleton coilSingleton2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Log.d("lpf","coilSingleton1 hashCode is"+coilSingleton1.hashCode());
        Log.d("lpf","coilSingleton2 hashCode is"+coilSingleton2.hashCode());

    }
}
```



**7.接口的注入**

* 准备一个接口

```java
public interface ICoilListener {
    void desc();
}
```



* 编写实现类，注意构造方法需要加@Inject注解

```java
public class CoilListenerImpl implements ICoilListener{

    @Inject
    public CoilListenerImpl() {
    }

    @Override
    public void desc() {
        Log.d("lpf","Kotlin 标配的图片加载框架，使用协程");
    }
}
```



* 编写Module，这个module是一个抽象类，方法也是抽象方法

```java
@InstallIn(ActivityComponent.class)
@Module
public abstract class CoilListenerModule {

    @Binds
    public abstract ICoilListener bindsICoilListener(CoilListenerImpl coilListener);

}
```



* 在Activity进行注入

```java
@AndroidEntryPoint
public class MainActivity extends AppCompatActivity {

    @Inject
    ICoilListener iCoilListener;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        iCoilListener.desc();
    }
}
```



总结：

* 本篇包含了对普通对象的注入
* 对单例的注入
* 对接口的注入

