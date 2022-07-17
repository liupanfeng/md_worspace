### Android Jetpack-DataBinding

Android标准化项目架构：MVVM+Jectpack

助力研发，本篇将对Jectpack 中的DataBinding进行简要分析



**1.什么是DataBinding？**

DataBinding是Google在2015年推出的组件库。Databinding支持双向绑定，可以大大减少绑定App逻辑于Layout的胶水代码。双向绑定，指的是将Model数据与界面绑定起来，当数据发生变化会直接体现在界面上，反过来界面发生变化也会同步到数据结构，使用DataBinding可以轻松实现MVVM模式。



**2.开启DataBinding**

在build.gradle配置

```groovy
dataBinding{
    enabled true
}
```

在Activity中加载

```
class DataBindingActivity : BaseActivity() {

    private val personInfo = PersonInfo()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityDataBindingBinding? = DataBindingUtil.
            setContentView(this,R.layout.activity_data_binding)

        personInfo.name.set("Tom")
        personInfo.pwd.set("123456")

        binding?.personInfo=personInfo

    }
}
```

在xml中进行绑定

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- DataBinding的编译标准 -->
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- 定义该View（布局）需要绑定的数据来源 -->

    <data>
        <variable
            name="personInfo"
            type="com.meishe.jetpackcollection.databinding.PersonInfo" />
    </data>



    <!-- 安卓认识的 常规布局 -->
    <!-- 布局常规代码如下 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity"
        android:orientation="vertical">

        <EditText
            android:id="@+id/et_name1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{personInfo.name}"
            />

        <EditText
            android:id="@+id/et_pwd1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{personInfo.pwd}"
            android:layout_marginBottom="60dp"
            />
            
    </LinearLayout>
</layout>
```

这样就实现了DataBinding

在Kotlin中的数据Bean中，由于不需要设置get、set方法，所以ObservableField进行包裹才行

```kotlin
Kotlinclass PersonInfo {

 val name : ObservableField<String> by lazy { ObservableField<String>() }
 val pwd : ObservableField<String> by lazy { ObservableField<String>() }

}
```



**3.实现双向绑定**

上面实现了单向绑定，实现双向绑定也很简单，只需要在Xml布局文件进行调整就好

```xml
<EditText
    android:id="@+id/et_name3"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:text="@={personInfo.name}"
    />

<EditText
    android:id="@+id/et_pwd3"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:text="@={personInfo.pwd}"
    android:layout_marginBottom="60dp"
    />
```

@{personInfo.pwd}更改为 @={personInfo.pwd}就可以实现另外另外一向的绑定操作。



**4.DataBinding中实现图片更改文本颜色等**

```kotlin
object CommonBindingAdapter {

    /**
     * 加载网络图片
     */
    @JvmStatic
    @BindingAdapter(value = ["imageUrl", "placeHolder"], requireAll = false)
    fun loadUrl(view: ImageView, url: String?, placeHolder: Drawable?) {
        Glide.with(view.context).load(url).placeholder(placeHolder).into(view)
    }

    /**
     * 控制显示隐藏
     */
    @BindingAdapter(value = ["visible"], requireAll = false)
    fun visible(view: View, visible: Boolean) {
        view.visibility = if (visible) View.VISIBLE else View.GONE
    }

    /**
     * 加载Drawable资源图片
     */
    @BindingAdapter(value = ["showDrawable", "drawableShowed"], requireAll = false)
    fun showDrawable(view: ImageView, showDrawable: Boolean, drawableShowed: Int) {
        view.setImageResource(if (showDrawable) drawableShowed else R.color.transparent)
    }

    /**
     * 动态设置文案颜色
     */
    @BindingAdapter(value = ["textColor"], requireAll = false)
    fun setTextColor(textView: TextView, textColorRes: Int) {
        textView.setTextColor(textView.resources.getColor(textColorRes))
    }

    /**
     * 通过Id加载图片资源
     */
    @BindingAdapter(value = ["imageRes"], requireAll = false)
    fun setImageRes(imageView: ImageView, imageRes: Int) {
        imageView.setImageResource(imageRes)
    }

    /**
     * 动态控制显示隐藏
     */
    @BindingAdapter(value = ["selected"], requireAll = false)
    fun selected(view: View, select: Boolean) {
        view.isSelected = select
    }
}
```

上面是DataBinding常用的动态加载资源的工具方法，收藏起来方便使用。



**5.DataBainding的原理**

构建之后，在这个路径下面会生成这个文件：

```shell
build/intermediates/data_binding_layout_info_type_merge/debug/out/activity_data_binding-layout.xml
```



```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Layout directory="layout" filePath="app\src\main\res\layout\activity_data_binding.xml"
    isBindingData="true" isMerge="false" layout="activity_data_binding"
    modulePackage="com.meishe.jetpackcollection" rootNodeType="android.widget.LinearLayout">
    <Targets>
        <Target tag="layout/activity_data_binding_0" view="LinearLayout">
            <Expressions />
            <location endLine="69" endOffset="18" startLine="16" startOffset="4" />
        </Target>
        <Target id="@+id/et_name2" tag="binding_1" view="EditText">
            <Expressions>
                <Expression attribute="android:text" text="personInfo.name">
                    <Location endLine="27" endOffset="44" startLine="27" startOffset="12" />
                    <TwoWay>false</TwoWay>
                    <ValueLocation endLine="27" endOffset="42" startLine="27" startOffset="28" />
                </Expression>
            </Expressions>
            <location endLine="28" endOffset="13" startLine="23" startOffset="8" />
        </Target>
        <Target id="@+id/et_pwd2" tag="binding_2" view="EditText">
            <Expressions>
                <Expression attribute="android:text" text="personInfo.pwd">
                    <Location endLine="34" endOffset="43" startLine="34" startOffset="12" />
                    <TwoWay>false</TwoWay>
                    <ValueLocation endLine="34" endOffset="41" startLine="34" startOffset="28" />
                </Expression>
            </Expressions>
            <location endLine="36" endOffset="13" startLine="30" startOffset="8" />
        </Target>
        <Target id="@+id/et_name3" tag="binding_3" view="EditText">
            <Expressions>
                <Expression attribute="android:text" text="personInfo.name">
                    <Location endLine="45" endOffset="45" startLine="45" startOffset="12" />
                    <TwoWay>true</TwoWay>
                    <ValueLocation endLine="45" endOffset="43" startLine="45" startOffset="29" />
                </Expression>
            </Expressions>
            <location endLine="46" endOffset="13" startLine="41" startOffset="8" />
        </Target>
        <Target id="@+id/et_pwd3" tag="binding_4" view="EditText">
            <Expressions>
                <Expression attribute="android:text" text="personInfo.pwd">
                    <Location endLine="52" endOffset="44" startLine="52" startOffset="12" />
                    <TwoWay>true</TwoWay>
                    <ValueLocation endLine="52" endOffset="42" startLine="52" startOffset="29" />
                </Expression>
            </Expressions>
            <location endLine="54" endOffset="13" startLine="48" startOffset="8" />
        </Target>
        <Target tag="binding_5" view="TextView">
            <Expressions>
                <Expression attribute="android:text" text="personInfo.name">
                    <Location endLine="60" endOffset="45" startLine="60" startOffset="12" />
                    <TwoWay>true</TwoWay>
                    <ValueLocation endLine="60" endOffset="43" startLine="60" startOffset="29" />
                </Expression>
            </Expressions>
            <location endLine="61" endOffset="13" startLine="57" startOffset="8" />
        </Target>
        <Target tag="binding_6" view="TextView">
            <Expressions>
                <Expression attribute="android:text" text="personInfo.pwd">
                    <Location endLine="66" endOffset="44" startLine="66" startOffset="12" />
                    <TwoWay>true</TwoWay>
                    <ValueLocation endLine="66" endOffset="42" startLine="66" startOffset="29" />
                </Expression>
            </Expressions>
            <location endLine="67" endOffset="13" startLine="63" startOffset="8" />
        </Target>
    </Targets>
    <Variables name="personInfo" declared="true"
        type="com.meishe.jetpackcollection.databinding.PersonInfo">
        <location endLine="10" endOffset="72" startLine="8" startOffset="8" />
    </Variables>
</Layout>
```

可以看出定义了多个Target标记，这些Target跟tag对应，tag跟布局文件的id是对应。



同时会生成

```shell
build/intermediates/incremental/mergeDebugResources/stripped.dir/layout/activity_data_binding.xml
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- DataBinding的编译标准 -->

                                                   

    <!-- 定义该View（布局）需要绑定的数据来源 -->

    
                 
                             
                                                                         
           



    <!-- 布局常规代码如下 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity"
        android:orientation="vertical" android:tag="layout/activity_data_binding_0" xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools">


        <EditText
            android:id="@+id/et_name2"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:tag="binding_1"          
            />

        <EditText
            android:id="@+id/et_pwd2"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:tag="binding_2"         
            android:layout_marginBottom="60dp"
            />



        <!-- 双向绑定   @= View  -->
        <EditText
            android:id="@+id/et_name3"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:tag="binding_3"           
            />

        <EditText
            android:id="@+id/et_pwd3"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:tag="binding_4"          
            android:layout_marginBottom="60dp"
            />


        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:tag="binding_5"           
            />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:tag="binding_6"          
            />

    </LinearLayout>
         
```

所以项目中的布局文件构建后会生成两个xml文件

* activity_data_binding-layout.xml
* activity_data_binding.xml

sdk层通过变量上面两个布局文件来解析数据的实现动态绑定数据的，其中tag是关键。





**DataBindingUtil.setContentView()**

主要就是调用Activity的setContentView设置布局，并且绑定对应的View。

```xml
public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
        int layoutId, @Nullable DataBindingComponent bindingComponent) {
    activity.setContentView(layoutId);
    View decorView = activity.getWindow().getDecorView();
    ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
    return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
}
```





**DataBindingUtil.bindToAddedViews()**

```xml
private static <T extends ViewDataBinding> T bindToAddedViews(DataBindingComponent component,
        ViewGroup parent, int startChildren, int layoutId) {
    final int endChildren = parent.getChildCount();
    final int childrenAdded = endChildren - startChildren;
    
    if (childrenAdded == 1) { //只有一个View
        final View childView = parent.getChildAt(endChildren - 1);
        return bind(component, childView, layoutId);
    } else { //多个View
        final View[] children = new View[childrenAdded];
        for (int i = 0; i < childrenAdded; i++) {
            children[i] = parent.getChildAt(i + startChildren);
        }
        return bind(component, children, layoutId); //绑定多个View
    }
}
```





**DataBindingUtil.bind()**

```java
static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View[] roots,
        int layoutId) {
    return (T) sMapper.getDataBinder(bindingComponent, roots, layoutId);
}
```

sMapper是DataBindingMapper的一个对象，实现类是DataBinderMapperImpl，DataBinderMapperImpl这个类是通过调用APT注解处理器生成的。

sMapper.getDataBinder方法，其实调用了MergedDataBinderMapper的addMepper方法

MergedDataBinderMapper也是继承DataBindingMapper的



**MergedDataBinderMapper.getDataBinder()**

```java
@Override
public ViewDataBinding getDataBinder(DataBindingComponent bindingComponent, View view,
        int layoutId) {
    for(DataBinderMapper mapper : mMappers) {
        ViewDataBinding result = mapper.getDataBinder(bindingComponent, view, layoutId);
        if (result != null) {
            return result;
        }
    }
    if (loadFeatures()) {
        return getDataBinder(bindingComponent, view, layoutId);
    }
    return null;
}
```

数据mMappers来源于DataBinderMapperImpl的构造器调用了addMapper方法得到的。

所以这里的数据就是com.meishe.jetpackcollection.DataBinderMapperImpl中的数据。

```shell
build/generated/ap_generated_sources/debug/out/com/meishe/jetpackcollection/DataBinderMapperImpl.java
```





**DataBinderMapperImpl.getDataBinder()**

```java
  @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      final Object tag = view.getTag();
      if(tag == null) {
        throw new RuntimeException("view must have a tag");
      }
      switch(localizedLayoutId) {
        case  LAYOUT_ACTIVITYDATABINDING: {
          if ("layout/activity_data_binding_0".equals(tag)) {
            return new ActivityDataBindingBindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for activity_data_binding is invalid. Received: " + tag);
        }
      }
    }
    return null;
  }

 @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View[] views, int layoutId) {
    if(views == null || views.length == 0) {
      return null;
    }
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      final Object tag = views[0].getTag();
      if(tag == null) {
        throw new RuntimeException("view must have a tag");
      }
      switch(localizedLayoutId) {
      }
    }
    return null;
  }
```

如果是底层的View，tag是layout/activity_data_binding_0，就会new ActivityDataBindingBindingImpl一个对象。



DataBinderMapperImpl的构造器在new处ActivityDataBindingBindingImpl之后会进行一些View的绑定操作。

通过tag取出来的view跟属性进行绑定操作。

```java
public ActivityDataBindingBindingImpl(@Nullable androidx.databinding.DataBindingComponent bindingComponent, @NonNull View root) {
    this(bindingComponent, root, mapBindings(bindingComponent, root, 7, sIncludes, sViewsWithIds));
}
```

调用mapBindings方法，这里的参数7就是说明布局文件包含7个节点，这个方法会返回Object[]数组。



**ViewDataBinding.mapBindings()**

```java
protected static Object[] mapBindings(DataBindingComponent bindingComponent, View root,
        int numBindings, IncludedLayouts includes, SparseIntArray viewsWithIds) {
    Object[] bindings = new Object[numBindings];
    mapBindings(bindingComponent, root, bindings, includes, viewsWithIds, true);
    return bindings;
}

 private static void mapBindings(DataBindingComponent bindingComponent, View view,
            Object[] bindings, IncludedLayouts includes, SparseIntArray viewsWithIds,
            boolean isRoot) {
        final int indexInIncludes;
        final ViewDataBinding existingBinding = getBinding(view);
		//判断view是否已经存在，如果已经绑定了直接返回
        if (existingBinding != null) {
            return;
        }
        //获取view的tag标签
        Object objTag = view.getTag();
        final String tag = (objTag instanceof String) ? (String) objTag : null;
        boolean isBound = false;
     	 //如果是跟布局，并且以layout开头的tag
        if (isRoot && tag != null && tag.startsWith("layout")) {
            final int underscoreIndex = tag.lastIndexOf('_');
            if (underscoreIndex > 0 && isNumeric(tag, underscoreIndex + 1)) {
                final int index = parseTagInt(tag, underscoreIndex + 1);
                //将布局对应的view放置在bindings数组里边
                if (bindings[index] == null) {
                    bindings[index] = view;
                }
                indexInIncludes = includes == null ? -1 : index;
                isBound = true;
            } else {
                indexInIncludes = -1;
            }
        } else if (tag != null && tag.startsWith(BINDING_TAG_PREFIX)) {
            int tagIndex = parseTagInt(tag, BINDING_NUMBER_START);
            if (bindings[tagIndex] == null) {
                bindings[tagIndex] = view;
            }
            isBound = true;
            indexInIncludes = includes == null ? -1 : tagIndex;
        } else {
            // Not a bound view
            indexInIncludes = -1;
        }
        if (!isBound) {
            final int id = view.getId();
            if (id > 0) {
                int index;
                if (viewsWithIds != null && (index = viewsWithIds.get(id, -1)) >= 0 &&
                        bindings[index] == null) {
                    bindings[index] = view;
                }
            }
        }

        if (view instanceof  ViewGroup) {
            final ViewGroup viewGroup = (ViewGroup) view;
            final int count = viewGroup.getChildCount();
            int minInclude = 0;
            for (int i = 0; i < count; i++) {
                final View child = viewGroup.getChildAt(i);
                boolean isInclude = false;
                if (indexInIncludes >= 0 && child.getTag() instanceof String) {
                    String childTag = (String) child.getTag();
                    if (childTag.endsWith("_0") &&
                            childTag.startsWith("layout") && childTag.indexOf('/') > 0) {
                        // This *could* be an include. Test against the expected includes.
                        int includeIndex = findIncludeIndex(childTag, minInclude,
                                includes, indexInIncludes);
                        if (includeIndex >= 0) {
                            isInclude = true;
                            minInclude = includeIndex + 1;
                            final int index = includes.indexes[indexInIncludes][includeIndex];
                            final int layoutId = includes.layoutIds[indexInIncludes][includeIndex];
                            int lastMatchingIndex = findLastMatching(viewGroup, i);
                            if (lastMatchingIndex == i) {
                                bindings[index] = DataBindingUtil.bind(bindingComponent, child,
                                        layoutId);
                            } else {
                                final int includeCount =  lastMatchingIndex - i + 1;
                                final View[] included = new View[includeCount];
                                for (int j = 0; j < includeCount; j++) {
                                    included[j] = viewGroup.getChildAt(i + j);
                                }
                                bindings[index] = DataBindingUtil.bind(bindingComponent, included,
                                        layoutId);
                                i += includeCount - 1;
                            }
                        }
                    }
                }
                if (!isInclude) {
                    mapBindings(bindingComponent, child, bindings, includes, viewsWithIds, false);
                }
            }
        }
    }

```



ActivityDataBindingBindingImpl的父类在ActivityDataBindingBinding

```shell
build/generated/data_binding_base_class_source_out/debug/out/com/meishe/jetpackcollection/databinding/ActivityDataBindingBinding.java
```

```java
public abstract class ActivityDataBindingBinding extends ViewDataBinding {
    //将xml布局文件的内容做了声明
  @NonNull
  public final EditText etName2;

  @NonNull
  public final EditText etName3;

  @NonNull
  public final EditText etPwd2;

  @NonNull
  public final EditText etPwd3;

   //将使用到的数据结构也做了声明
  @Bindable
  protected PersonInfo mPersonInfo;

  protected ActivityDataBindingBinding(Object _bindingComponent, View _root, int _localFieldCount,
      EditText etName2, EditText etName3, EditText etPwd2, EditText etPwd3) {
    super(_bindingComponent, _root, _localFieldCount);
    this.etName2 = etName2;
    this.etName3 = etName3;
    this.etPwd2 = etPwd2;
    this.etPwd3 = etPwd3;
  }

  public abstract void setPersonInfo(@Nullable PersonInfo personInfo);

  @Nullable
  public PersonInfo getPersonInfo() {
    return mPersonInfo;
  }

  @NonNull
  public static ActivityDataBindingBinding inflate(@NonNull LayoutInflater inflater,
      @Nullable ViewGroup root, boolean attachToRoot) {
    return inflate(inflater, root, attachToRoot, DataBindingUtil.getDefaultComponent());
  }

  /**
   * This method receives DataBindingComponent instance as type Object instead of
   * type DataBindingComponent to avoid causing too many compilation errors if
   * compilation fails for another reason.
   * https://issuetracker.google.com/issues/116541301
   * @Deprecated Use DataBindingUtil.inflate(inflater, R.layout.activity_data_binding, root, attachToRoot, component)
   */
  @NonNull
  @Deprecated
  public static ActivityDataBindingBinding inflate(@NonNull LayoutInflater inflater,
      @Nullable ViewGroup root, boolean attachToRoot, @Nullable Object component) {
    return ViewDataBinding.<ActivityDataBindingBinding>inflateInternal(inflater, R.layout.activity_data_binding, root, attachToRoot, component);
  }

  @NonNull
  public static ActivityDataBindingBinding inflate(@NonNull LayoutInflater inflater) {
    return inflate(inflater, DataBindingUtil.getDefaultComponent());
  }

  /**
   * This method receives DataBindingComponent instance as type Object instead of
   * type DataBindingComponent to avoid causing too many compilation errors if
   * compilation fails for another reason.
   * https://issuetracker.google.com/issues/116541301
   * @Deprecated Use DataBindingUtil.inflate(inflater, R.layout.activity_data_binding, null, false, component)
   */
  @NonNull
  @Deprecated
  public static ActivityDataBindingBinding inflate(@NonNull LayoutInflater inflater,
      @Nullable Object component) {
    return ViewDataBinding.<ActivityDataBindingBinding>inflateInternal(inflater, R.layout.activity_data_binding, null, false, component);
  }

  public static ActivityDataBindingBinding bind(@NonNull View view) {
    return bind(view, DataBindingUtil.getDefaultComponent());
  }

  /**
   * This method receives DataBindingComponent instance as type Object instead of
   * type DataBindingComponent to avoid causing too many compilation errors if
   * compilation fails for another reason.
   * https://issuetracker.google.com/issues/116541301
   * @Deprecated Use DataBindingUtil.bind(view, component)
   */
  @Deprecated
  public static ActivityDataBindingBinding bind(@NonNull View view, @Nullable Object component) {
    return (ActivityDataBindingBinding)bind(component, view, R.layout.activity_data_binding);
  }
}
```

