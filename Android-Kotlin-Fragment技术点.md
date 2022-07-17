### Android-Kotlin-Fragment技术点

**1.Fragment定义**

Fragment是一种可以嵌入在Activity当中的UI片段，它能够让程序更加合理的利用屏幕空间，可以把它理解成一个迷你型的Activity



**2.Fragment声明周期**

onAttach()->onCreate()->onCreateView()->onActivityCreate()->onStart()->onResume()->onPause()->onStop()->onDestroyView()->onDestroy()->onDetach()

* onAttach():当Fragment和Activity建立关联的时候调用
* onCreateView():为Fragment创建视图的时候调用
* onActivityCreate():确保与Fragment相关联的Activity已经创建完毕时回调
* onDestroyView():当与Fragment相关联的视图被移除的时候调用
* onDetach():当Fragment和Activity解除关联的时候调用



**3.Fragment的使用**

```kotlin
<fragment android:id="@+id/leftFragment"
          android:layout_width="0dp"
          android:layout_height="match_parent"
          android:layout_weight="1"
          android:name="com.test.kotlin_test.fragment.LeftFragment"
/>
```

通过fragment标签直接在布局文件添加，通过name来指定使用那个fragment，这个方式直接固定在布局文件了，如果需要更换不是很方便。



```kotlin
private fun replaceFragment(rightFragment: Fragment) {
    val fragmentManager = supportFragmentManager
    val transaction = fragmentManager.beginTransaction()
    transaction.apply {
        replace(R.id.rightFragment, rightFragment)
        addToBackStack(null)
        commit()
    }
}
```

动态替换添加Fragment，通过addToBackStack方法可以将事务添加到返回栈里边，点击返回会回到前面一个fragment的状态，需要传一个名字描述返回栈的状态，根据需求如果没有特殊要求可以传null



**4.Fragment和Activity交互**

```Kotlin
val fragmentActivity = activity as FragmentActivity
```

在Fragment中通过getActivity方法获取到的就是包含它的Activity实例，强转之后就可以执行Activity里边的方法了



```Kotlin
val leftFragment = supportFragmentManager.findFragmentById(R.id.leftFragment) as LeftFragment
```

在Activity执行findFragmentById可以得到加载的fragment实例，便可以方便的执行fragment里边的方法了



**5.Activity向Fragment传递参数**

```kotlin
//通过Bundle传递参数，并把bundle设置给argument
val bundle=Bundle()
bundle.putString(LeftFragment.KEY_PARAM,"test")
rightFragment.arguments=bundle

//在Fragment的onCreate方法中得到参数
   override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val param = arguments?.getString(KEY_PARAM)
    }
```

Fragment有个argument参数，可以通过这个来给Fragment传递参数。



**6.动态加载布局**

前面了解了动态加载Fragment的方式，程序也是可以根据设备的分辨率或者屏幕的大小来加载布局文件，实现这个就需要了解限定符了



**large限定符**

新建res 下面layout-large文件夹，这个文件下面新建一个名称一样的布局文件activity_fragment.xml，这个里边包含左右两个Fragment，layout里边activity_fragment.xml包含一个Fragment。

将程序部署到平板模拟器上显示了layout-large里边的布局文件

将程序部署在手机上显示了layout里边的布局文件



**smallest-width最小宽度限定符**

对屏幕的宽度指定一个最小值，然后以这个最小值作为临界值

可以在不同的宽度分辨率下面指定不同的布局文件，做到完美的布局文件适配

layout-sw500dp 

layout-sw400dp



