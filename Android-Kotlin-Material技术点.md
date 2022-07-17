### Android-Kotlin-Material Design技术点

**1.Material design 定义**

Material design 是由google的设计工程师基于优秀的设计原则，结合丰富的创意和科学技术所开发的一套全新的界面设计语言，包含了视觉、运动、互动特效等特性。



**2.Toolbar的使用**

Toolbar不仅继承了ActionBar的所有功能，而且灵活性很高，可以配合其他控件完成一些Material Design的效果。

创建项目的时候默认会使用ActionBar，这个是在AppTheme中定义的。如果想使用toolbar需要指定不带ActionBar的主题。一般使用Theme.AppCompat.NoActionBar或者Theme.Appcompat.Light.NoActionBar。

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        tools:context=".MaterialActivity">

    <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolBar"
            android:layout_width="match_parent"
             android:layout_height="?attr/actionBarSize"
             android:background="@color/colorPrimary"
             android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
             app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
    />

</FrameLayout>
```

给一个界面的布局文件指定ToolBar，然后在Activity中设置setSupportActionBar(toolBar)，就可以显示出来标题了，看起来跟ActionBar下效果一样，但是已经不是ActionBar了，是ToolBar了。

指定ToolBar的文案使用label标签

```kotlin
<activity android:name=".MaterialActivity"
    android:label="美女">
```



现在看起来很单调，设置一个menu给ToolBar，在Res文件下新建menu文件夹，创建toolbar.xml文件

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android" xmlns:app="http://schemas.android.com/apk/res-auto">

    <item android:id="@+id/backup"
          android:icon="@mipmap/ic_backup"
          android:title="Backup"
          app:showAsAction="always"/>

    <item android:id="@+id/delete"
          android:icon="@mipmap/ic_delete"
          android:title="Delete"
          app:showAsAction="ifRoom"/>

    <item android:id="@+id/settings"
          android:icon="@mipmap/ic_settings"
          android:title="settings"
          app:showAsAction="never"/>

</menu>
```

在activity中重写菜单的两个方法，一个是onCreateOptionsMenu这个是创建菜单，返回上面的菜单就好，另外一个是onOptionsItemSelected这个是菜单点击回调。

```kotlin
override fun onCreateOptionsMenu(menu: Menu?): Boolean {
    menuInflater.inflate(R.menu.toolbar,menu)
    return true
}

override fun onOptionsItemSelected(item: MenuItem): Boolean {
    when(item.itemId){
        R.id.backup -> showToast("backup")
        R.id.delete -> showToast("delete")
        R.id.settings -> showToast("settings")
    }

    return true
}
```



**3.滑动菜单DrawerLayout**

DrawerLayout是一个滑动菜单，不用的时候可以隐藏起来，使用的使用把它通过滑动的方式显示出来。节省了屏幕控件又带有非常好的动画效果，是Material Design中的推荐做法。

DrawerLayout 允许有两个直接子控件，第一个是显示的主屏幕的内容，另外一个是子菜单的内容，实例如下：

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.drawerlayout.widget.DrawerLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:id="@+id/drawerLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
       >

    <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            tools:context=".materila.MaterialActivity">

        <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolBar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                android:background="@color/colorPrimary"
                android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        />

    </FrameLayout>

    <TextView android:layout_width="match_parent"
              android:text="this is menu"
              android:background="#fff"
              android:textSize="29sp"
              android:layout_gravity="start"
              android:layout_height="match_parent"/>

</androidx.drawerlayout.widget.DrawerLayout>
```

**注意这个必须指定： android:layout_gravity="start"**，这样就可以通过右滑动来显示这个菜单了。由于用户可能不知道有这样一个菜单，一般还需要增加一个导航按钮。



```kotlin
 override fun onCreate(savedInstanceState: Bundle?) {
       ...
        setSupportActionBar(toolBar)
        supportActionBar?.let {
            it.setDisplayHomeAsUpEnabled(true)
            it.setHomeAsUpIndicator(R.mipmap.ic_menu)
        }
    }


    override fun onOptionsItemSelected(item: MenuItem): Boolean {
        when(item.itemId){
            android.R.id.home -> {
                drawerLayout.openDrawer(GravityCompat.START)
            }
         ...
        }

        return true
    }
}
```

这样就设置好了Home导航按钮，以及点击事件，ToolBar最左侧的按钮就是Home按钮，默认的图片是一个返回箭头



**4.NavigationView**

这个控件非常适合做滑动菜单的内容，效果非常好，实现也非常简单。

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <group android:checkableBehavior="single">
        <item
                android:id="@+id/navCall"
                android:icon="@mipmap/nav_call"
                android:title="Call"/>

        <item android:id="@+id/navFriends"
              android:icon="@mipmap/nav_friends"
              android:title="Friends"/>

        <item android:id="@+id/navLocation"
              android:icon="@mipmap/nav_location"
              android:title="Location"/>

        <item android:id="@+id/navMail"
              android:icon="@mipmap/nav_mail"
              android:title="Mail"/>

        <item android:id="@+id/navTask"
              android:icon="@mipmap/nav_task"
              android:title="Tasks"/>
    </group>

</menu>
```

创建NavMenu



```kotlin
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                xmlns:app="http://schemas.android.com/apk/res-auto"
                android:orientation="vertical"
                android:layout_width="match_parent"
                android:padding="10dp"
                android:background="@color/colorPrimary"
                android:layout_height="180dp">

    <de.hdodenhof.circleimageview.CircleImageView
            android:id="@+id/iconImage"
            app:civ_border_width="2dp"
            app:civ_border_color="#fff"
            android:layout_width="70dp"
            android:src="@mipmap/nav_icon"
            android:layout_centerInParent="true"
            android:layout_height="70dp"/>

    <TextView
            android:id="@+id/mailText"
            android:layout_width="wrap_content"
            android:layout_alignParentBottom="true"
            android:textColor="#fff"
            android:textSize="14sp"
            android:text="她在滑雪场滑雪呢"
            android:layout_height="wrap_content"/>

    <TextView
            android:id="@+id/userText"
            android:layout_width="wrap_content"
            android:textColor="#fff"
            android:textSize="14sp"
            android:layout_height="wrap_content"
        android:layout_above="@+id/mailText"
              android:text="刘若兮和雪人"
    />

</RelativeLayout>
```

创建Nav_Header



```kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.drawerlayout.widget.DrawerLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:id="@+id/drawerLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
       >

   ...

    <com.google.android.material.navigation.NavigationView
            android:id="@+id/navView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_gravity="start"
            app:menu="@menu/nav_menu"
            app:headerLayout="@layout/nav_header"
    />

</androidx.drawerlayout.widget.DrawerLayout>
```

将nav_menu和nav_header添加到NavigationView中。



```
override fun onCreate(savedInstanceState: Bundle?) {
   ...
    navView.setCheckedItem(R.id.navCall)  //默认选中某个item
    navView.setNavigationItemSelectedListener {
        when(it.itemId){
            R.id.navCall-> showToast("Call")
            R.id.navLocation-> showToast("Location")
        }
        drawerLayout.closeDrawers()
        true
    }
}
```

设置navicationView的点击事件以及默认选中



**5.悬浮按钮FloatingActionButton**

这个是底部带阴影的悬浮按钮，具有立体的效果，默认使用colorAccent作为按钮的颜色。

```Kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.drawerlayout.widget.DrawerLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:id="@+id/drawerLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
       >

    <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            tools:context=".materila.MaterialActivity">

      ...

        <com.google.android.material.floatingactionbutton.FloatingActionButton
                android:id="@+id/fab"
                android:layout_gravity="bottom|end"
                android:layout_margin="15dp"
                android:src="@mipmap/ic_done"
                app:elevation="4dp"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>

    </FrameLayout>

...

</androidx.drawerlayout.widget.DrawerLayout>
```

在布局文件中添加这个控件，可以通过elcvation这个属性设置高度值，高度值越大投影范围越小。

backgroundTint 通过这个属性可以更改默认背景颜色





```kotlin
fab.setOnClickListener {
    showToast("FAB click")
}
```

给悬浮按钮设置点击事件



**6.SnackBar**

这个控件更Toast类似，但是并不是Toast的替代品，它有不同的应用场景，SnackBar给用户提示的同时还可以提供了一个交互的设计。

```kotlin
fab.setOnClickListener {
    Snackbar.make(it,"Data deleted",Snackbar.LENGTH_LONG).setAction("Done"){
        showToast("confirm delete data")
    }.show()
}
```

可见用法跟Toast类似，增加了一个setAction的方法。SnackBar从底部弹出，并带有动画效果，会自动从底部消失。



**7.CoordinatorLayout**

上面在弹出SnackBar会遮挡FloatActionButton，这个时候就可以通过使用CoordinatorLayout来解决这个小bug了，CoordinatorLayout这个是FrameLayout的加强版。这个容器扩展了对子控件的事件监听，并自动帮我们做出最合理的响应。

修复FloatActionButton被遮挡的问题，只需要将FrameLayout替换成CoordinatorLayout就可以，替换之后没有任何其他的问题。弹出SnackBar之后，悬浮按钮会自动向上移动。SnackBar消失悬浮按钮会自动的回到原来的位置，整个过程都是带有动画的非常丝滑。



**8.MaterialCardView与RecyclerView**

MaterialCardView是卡片式布局效果的重要控件，它也是一个FrameLayout，只是额外提供了圆角和阴影效果。

RecyclerView是ListView的加强版本。

```Kotlin
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent" xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_margin="6dp"
        app:cardCornerRadius="4dp"
        android:layout_height="wrap_content">
    <LinearLayout android:layout_width="match_parent" android:layout_height="wrap_content" android:orientation="vertical" >
        <ImageView android:id="@+id/girlImage" android:layout_width="match_parent" android:layout_height="100dp" android:scaleType="centerCrop"/>
        <TextView android:layout_width="wrap_content" android:layout_height="wrap_content" android:layout_gravity="center_horizontal"
                  android:layout_margin="5dp"
                  android:textSize="15sp"
                  android:id="@+id/girlName"
        />
    </LinearLayout>

</com.google.android.material.card.MaterialCardView>
```

创建卡片Item布局



```Kotlin
class GirlAdapter(private val context:Context, private val girlList:List<BeautyGirl>): RecyclerView.Adapter<GirlAdapter.GirlHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): GirlHolder {
        val view:View = LayoutInflater.from(context).inflate(R.layout.girl_item,parent,false)
        return GirlHolder(view)
    }

    override fun getItemCount(): Int {
        return girlList.size
    }

    override fun onBindViewHolder(holder: GirlHolder, position: Int) {
        val girl= girlList[position]
        holder.girlName.text = girl.name
        Glide.with(context).load(girl.imageId).into(holder.girlImage)
    }

    class GirlHolder(view : View) : RecyclerView.ViewHolder(view){
        var girlImage:ImageView = view.findViewById(R.id.girlImage)
        var girlName:TextView = view.findViewById(R.id.girlName)
    }

}
```

新建RecyclerView使用的适配器，通过构造方法设置数据



```Kotlin
private fun initRecyclerView() {
    val girls= mutableListOf(BeautyGirl("美女1",R.mipmap.g1),BeautyGirl("美女2",R.mipmap.g2),BeautyGirl("美女3",R.mipmap.g3),
        BeautyGirl("美女4",R.mipmap.g4),BeautyGirl("美女5",R.mipmap.g5),BeautyGirl("美女6",R.mipmap.g6))

    val layoutManger=GridLayoutManager(this,2)
    recyclerView.layoutManager=layoutManger
    val adapter=GirlAdapter(this,girls)
    recyclerView.adapter=adapter
}
```

初始化RecyclerView，并设置数据，这样卡片样式的view就能显示到手机上了，整个列表优雅漂亮。



**9.AppBarLayout**

在上面展示列表的时候，发现列表遮挡住了ToolBar。传统的解决方法是，让RecyclerView设置marginTop值大小是Toolbar的高度，目前使用的外层容器时CoordinatorLayout，借助AppBarLayout这个控件就能修复这个问题，使用这个控件优势是：AppBarLayout具备了Material Design的设计理念。

```kotlin
<androidx.coordinatorlayout.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
      >

    <com.google.android.material.appbar.AppBarLayout android:layout_width="match_parent"
                                                     android:layout_height="wrap_content">
        <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolBar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                android:background="@color/colorPrimary"
                android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        />
    </com.google.android.material.appbar.AppBarLayout>



    <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/recyclerView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_behavior="@string/appbar_scrolling_view_behavior"
    />

...

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

改动也不大，只需要**将ToolBar移到AppBarLayout里边**，然后将**RecyclerView指定behavior行为**就可以，其中appbar_scrolling_view_behavior在material 库里边。

**当RecyclerView滚动的时候AppBarLayout就能收到滑动事件了**，这样并没有看到Material Design效果。是因为AppBarLayout收到滑动事件，并没有处理滑动事件。

给ToolBar添加一个app:layout_scrollFlags,并设置值为**sroll|enterAlways|snap**

**scroll表示：当RecyclerView滚动的时候，Toolbar会随着滚动，并实现隐藏。**

**enterAlways表示：当RecyclerView向下滚动的时候ToolBar会向下滚动并重新显示出来**

**snap表示：当Toolbar还没有完全隐藏或者显示的时候，会根据当前滚动的距离，自动选择显示还是隐藏。**



**10.实现下拉刷新SwipeRefreshLayout**

列表下拉刷新，有各种各样的实现版本，风格也格式各样，下面介绍使用Google官方提供的SwipeRefreshLayout实现下拉刷新。

```Kotlin
implementation 'androidx.swiperefreshlayout:swiperefreshlayout:1.0.0'
```

增加这个依赖



```kotlin
<androidx.swiperefreshlayout.widget.SwipeRefreshLayout
        android:id="@+id/swipeRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"
>

    <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/recyclerView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_behavior="@string/appbar_scrolling_view_behavior"
    />
</androidx.swiperefreshlayout.widget.SwipeRefreshLayout>
```

将RecyclerView添加到SwipeRefreshLayout，注意将前面提到的behavior添加到SwipeRefreshLayout上面，这样就自动拥有了下拉刷新的功能了。



```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
   ...
    swipeRefreshLayout.setColorSchemeResources(R.color.colorPrimary)
    swipeRefreshLayout.setOnRefreshListener {
        refreshGirls()
    }
}

    fun refreshGirls(){
        thread {
            Thread.sleep(2000)
            runOnUiThread{
                initGirls()
                adapter.setData(girls)
                swipeRefreshLayout.isRefreshing=false
            }
        }
    }
```

增加具体的刷新代码逻辑



**11.折叠标题栏CollapsingToolbarLayout**

CollapsingToolbarLayout是一个作用于Toolbar基础上的布局文件，可以让toolbar的效果变的丰富多彩，这个控件不能独立存在，只能作为AppBarLayout的子布局，而AppBarLayout又必须是CoordinatorLayout的子布局。

创建一个详情页面，布局文件如下

```kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".materila.GirlDetailActivity">
    <com.google.android.material.appbar.AppBarLayout android:id="@+id/appBar"
                                                     android:layout_width="match_parent"
                                                     android:layout_height="250dp">
        <com.google.android.material.appbar.CollapsingToolbarLayout android:id="@+id/collapsingToolBar"
                                                                    android:layout_width="match_parent"
                                                                    android:layout_height="match_parent"
                                                                    android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
                                                                    app:contentScrim="@color/colorPrimary"
                                                                    app:layout_scrollFlags="scroll|exitUntilCollapsed">
            <ImageView
                    android:id="@+id/girlImage"
                    android:layout_width="match_parent"
                    android:scaleType="centerCrop"
                    android:layout_height="match_parent"
                    app:layout_collapseMode="parallax"
            />

            <androidx.appcompat.widget.Toolbar
                    android:id="@+id/toolBar"
                    android:layout_width="match_parent"
                    android:layout_height="?actionBarSize"
                    app:layout_collapseMode="pin"
            />

        </com.google.android.material.appbar.CollapsingToolbarLayout>

    </com.google.android.material.appbar.AppBarLayout>

    <androidx.core.widget.NestedScrollView android:layout_width="match_parent"
                                           android:layout_height="match_parent"
                                           app:layout_behavior="@string/appbar_scrolling_view_behavior">
        <LinearLayout android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:orientation="vertical">
            <com.google.android.material.card.MaterialCardView android:layout_width="match_parent"
                                                               android:layout_height="wrap_content"
                                                               android:layout_margin="15dp" app:cardCornerRadius="4dp">
                <TextView android:id="@+id/content"
                          android:layout_width="wrap_content"
                          android:layout_height="wrap_content"
                          android:layout_margin="10dp"
                />
            </com.google.android.material.card.MaterialCardView>
        </LinearLayout>

    </androidx.core.widget.NestedScrollView>

    <com.google.android.material.floatingactionbutton.FloatingActionButton android:layout_width="wrap_content"
                                                                           android:layout_height="wrap_content"
                                                                           android:layout_margin="16dp"
                                                                           android:src="@mipmap/ic_comment"
                                                                           app:layout_anchor="@id/appBar"
                                                                           app:layout_anchorGravity="bottom|end"
    />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

1.最外层的容器是CoordinatorLayout，由于AppBarLayout父容器必须是CoordinatorLayout

2.toolBar部分最外层是AppBarLayout，里边是CollapsingToolBarLayout，CollapsingToolBarLayout这个控件设置了 **3.app:contentScrim="@color/colorPrimary" 这个表示在趋于折叠状态或者折叠之后的颜色**

**4.exitUntilCollapsed：表示当collapsingToolBarLayout随着折叠之后就保留在界面上了，不再移出屏幕**

5.app:layout_collapseMode="parallax" 这个表示**折叠过程中的样式**，设置为**parallax表示折叠过程中产生一定的错位效果，错位视觉差。**

6.NestedScrollView相比于ScrollView增加了**嵌套响应滚动事件**，并给这个控件指定了一个布局行为，内部只允许一个子控件

7.最后添加了一个悬浮按钮，

* layout_anchor：指定一个锚点，这里锚点是AppBarLayout
* app:layout_anchorGravity="bottom|end" ：相对于锚点所在的位置



```Kotlin
class GirlDetailActivity : AppCompatActivity() {

    companion object{
        const val GIRL_NAME="girl_name"
        const val GIRL_IMAGE="girl_image"
    }
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_gril_detail)
        val girlName=intent.getStringExtra(GIRL_NAME)?:""
        val girlImageId=intent.getIntExtra(GIRL_IMAGE,0)
        setSupportActionBar(toolBar)
        supportActionBar?.setDisplayHomeAsUpEnabled(true)
        collapsingToolBar.title=girlName
        Glide.with(this).load(girlImageId).into(girlImage)
        content.text = initContent(girlName)

    }

    private fun initContent(girlName:String): String =girlName.repeat(500)

    override fun onOptionsItemSelected(item: MenuItem): Boolean {
        when(item.itemId){
            android.R.id.home->{
                finish()
                return true
            }
        }
        return super.onOptionsItemSelected(item)
    }

}
```

这里是详情页面的代码逻辑，设置了collapsingToobar的标题等，启动了HomeButton重写了menu的点击事件



```Kotlin
class GirlAdapter(private val context:Context): RecyclerView.Adapter<GirlAdapter.GirlHolder>() {

    private   val mGirlList:MutableList<BeautyGirl> =ArrayList()

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): GirlHolder {
        val view:View = LayoutInflater.from(context).inflate(R.layout.girl_item,parent,false)
        val holder=GirlHolder(view)
        holder.itemView.setOnClickListener {
            val position=holder.adapterPosition
            val girl=mGirlList[position]
            val intent= Intent(context,GirlDetailActivity::class.java).apply {
                putExtra(GirlDetailActivity.GIRL_NAME,girl.name)
                putExtra(GirlDetailActivity.GIRL_IMAGE,girl.imageId)
            }
            context.startActivity(intent)
        }
        return holder
    }

    fun setData(girlList:MutableList<BeautyGirl>?){
        mGirlList.clear()
        girlList?.let { mGirlList.addAll(it) }
        notifyDataSetChanged()
    }

    override fun getItemCount(): Int {
        return mGirlList.size
    }

    override fun onBindViewHolder(holder: GirlHolder, position: Int) {
        val girl= mGirlList[position]
        holder.girlName.text = girl.name
        Glide.with(context).load(girl.imageId).into(holder.girlImage)
    }

    class GirlHolder(view : View) : RecyclerView.ViewHolder(view){
        var girlImage:ImageView = view.findViewById(R.id.girlImage)
        var girlName:TextView = view.findViewById(R.id.girlName)
    }
   }
```

增加了RecyclerView的点击事件，并将参数传递给了详情页面，这样详情页面的标题栏就有折叠的效果，非常漂亮。



**12.背景图跟系统状态栏视觉不搭配的问题**

详情页面的背景图片跟系统状态栏看起来总是不搭配，可以通过设置android:fitsSystemWindows="true"，设置为true表示该控件出现了系统状态栏里边。

注意CoordinatorLayout、AppBarLayout、CollapsingToolbarLayout以及ImageView都需要设置

```Kotlin
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="true"
        tools:context=".materila.GirlDetailActivity">
    <com.google.android.material.appbar.AppBarLayout android:id="@+id/appBar"
                                                     android:layout_width="match_parent"
                                                     android:fitsSystemWindows="true"
                                                     android:layout_height="250dp">
        <com.google.android.material.appbar.CollapsingToolbarLayout android:id="@+id/collapsingToolBar"
                                                                    android:layout_width="match_parent"
                                                                    android:layout_height="match_parent"
                                                                    android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
                                                                    app:contentScrim="@color/colorPrimary"
                                                                    android:fitsSystemWindows="true"
                                                                    app:layout_scrollFlags="scroll|exitUntilCollapsed">
            <ImageView
                    android:id="@+id/girlImage"
                    android:layout_width="match_parent"
                    android:scaleType="centerCrop"
                    android:layout_height="match_parent"
                    app:layout_collapseMode="parallax"
                    android:fitsSystemWindows="true"
            />

            <androidx.appcompat.widget.Toolbar
                    android:id="@+id/toolBar"
                    android:layout_width="match_parent"
                    android:layout_height="?actionBarSize"
                    app:layout_collapseMode="pin"
            />

        </com.google.android.material.appbar.CollapsingToolbarLayout>

    </com.google.android.material.appbar.AppBarLayout>

...

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```



除此之外还需要将系统状态栏设置为透明状态，给这个详情页面指定下面这个属性即可，齐刘海等屏幕都有效果，但是需要Android5.0或者以上的系统才可以。

```kotlin
<style name="girlActivityTheme" parent="AppTheme">
    <item name="android:statusBarColor">@android:color/transparent</item>
</style>
```

这样系统状态栏跟背景图片就融合到一起了，解决了背景图片跟系统状态栏颜色不搭配的问题。





