### Android Jetpack-ViewModel

Jetpack代表了Android原生开发未来的方向。

Android标准化项目架构：MVVM+Jectpack。

助力研发，本篇将对Jectpack 中的ViewModel进行简要分析。



**1.ViewModel用来做什么？**

ViewModel:可以帮助Activity分担一部分工作，它是专门用于存放与界面相关的数据的。只要是界面上能看得到的数据，它的相关变量都应该存放在ViewModel中，而不是Activity中，这样可以在一定程度上减少Activity中的逻辑。



**2.ViewModel的初始化**

```kotlin
class MainViewModel:ViewModel() {

    var number : String = "Tom"
}
```

定义一个ViewModel



```kotlin
// mViewModel = MainViewModel()  不能直接通过new的方式实例化
```

不用通过直接new的方式初始化，因为如果能这样写，系统不可控了



```kotlin
// ViewModelProviders.of()
```

可以看到这个被标记为废弃了，因为发展非常快，不容易扩展，这个方式也不推荐。



```kotlin
//初始化ViewModel的正确姿势
mViewModel=ViewModelProvider(this).get(MainViewModel::class.java)
```

这个是正确的初始化的方式。



通过继承AndroidViewModel来实现ViewModel

```kotlin
class MainViewModel(application: Application):AndroidViewModel(application) {

 	val mContext:Context=application
	val phone by lazy { MutableLiveData<String>() }
}
```

会带一个上下文Context



初始化

```kotlin
mMainViewModel=ViewModelProvider(this,
    ViewModelProvider.AndroidViewModelFactory(application)).get(MainViewModel::class.java)
```



