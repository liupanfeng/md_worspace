### Android Jetpack-ViewBinding

Android标准化项目架构：MVVM+Jectpack

助力研发，本篇将对Jectpack 中的ViewBinding进行简要分析



**1.ViewBinding是什么？**

ViewBinding可以理解为轻量级的DataBinding，使用ViewBinding之后，不再需要使用findViewById等，可以大幅度提升开发效率。



**2.ViewBinding是通过APT注解处理器实现的吗？**

ViewBinding不是通过注解处理器实现的，是一个即时的小组件，只要在xml布局声明控件之后，马上就能引用到，不需要调用构建，**是通过Gradle插件技术实现的**。



**3.ViewBinding的使用**

```groovy
// 启用ViewBinding
viewBinding {
    enabled true
}
```

跟启动DataBinding类似，直接在build.gradle添加这个就启动了



**4.在Activity中使用**

```kotlin
class LiveDataActivity : BaseActivity() {

    var mBinding:ActivityLiveDataBinding?=null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        mBinding= ActivityLiveDataBinding.inflate(layoutInflater)
        setContentView(mBinding?.root)

        mBinding?.button?.setOnClickListener{
            startService(Intent(this,MSService::class.java))
            Toast.makeText(LiveDataActivity@this,"启动服务后台模拟后台推送数据",Toast.LENGTH_LONG).show()
        }
     }
```

控件直接通过ViewBinding就可以拿到，非常方便。



**5.在Frament同样可以使用**

```kotlin
public class MSFragment extends Fragment {

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {

        FragmentMSBinding fragmentBlankBinding = FragmentBlankBinding.inflate(getLayoutInflater());
        fragmentMSBinding.ff1.setText("viewbinding");

        // Inflate the layout for this fragment
        // return inflater.inflate(R.layout.fragment_blank, container, false);

        return fragmentMSkBinding.getRoot();
    }
```



在项目中直接使用ViewBinding可以大幅提高研发效率。

