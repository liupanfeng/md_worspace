### Android Coil 图片加载库

[TOC]

#### 1.Coil库的特点

* Coil可以配合Kotlin语言的协程实现图片加载，非常适合在Kotlin/Android项目中使用。
* 加载性能好，底层使用协程实现。
* 具有缓存管理（MemCache、DiskCache）、动态采样（Dynamic image sampling）、 加载中暂停/终止等功能有助于提高图片加载效率
* 轻量级（其包体积与Picasso相当，显著低于Glide和Fresco，仅仅只有1500个方 法，但是在功能上却非常强大：支持透明度、圆角、高斯模糊、以及通过Transformation自定义一些高级的功能
* .简单易用（配合Kotlin扩展方法等语法优势，API简单易用），函数式编程思想。
* 5.技术先进（基于Coroutine、OkHttp、Okio、AndroidX等先端技术开发）



#### 2.Coil框架依赖

```groovy
implementation("io.coil-kt:coil:1.1.1")

// 如果想要显示 Gif、SVG、视频帧等类型的图片，则需要额外引入对应的支持库：
implementation("io.coil-kt:coil-gif:1.0.0")
implementation("io.coil-kt:coil-svg:1.0.0")
implementation("io.coil-kt:coil-video:1.0.0")
```



#### 3.Coil的使用

##### 3.1简单使用

```kotlin
//函数式编程思想
imageView.load(imageUrl1);  //直接加载图片
imageView.load(R.mipmap.g4) //加载资源素材
```

直接使用load传入资源链接就好。



#### 3.2实现淡入淡出，并实现圆形效果

```kotlin
        imageView.load(imageUrl1){
            crossfade(true)     //开启淡入淡出
            crossfade(3000)
            placeholder(R.mipmap.placeholder)   //添加占位图
            transformations(CircleCropTransformation())  //图片变换  圆形
        }
```



#### 3.3包含错误占位图

```kotlin
imageView.load(errorUrl){
    crossfade(true)     //开启淡入淡出
    crossfade(3000)
    error(R.mipmap.error)   //加载失败的情况
    transformations(CircleCropTransformation())  //图片变换
}
```



#### 3.4轻松实现圆角

```kotlin
imageView.load(imageUrl1){
    crossfade(true)     //开启淡入淡出
    crossfade(3000)
    placeholder(R.mipmap.placeholder)   //添加占位图
    transformations(RoundedCornersTransformation(topRight = 30f, topLeft = 30f, bottomLeft = 30f, bottomRight = 30f))  //图片变换  加圆角
}
```



##### 3.5高斯模糊效果

```kotlin
imageView.load(imageUrl2){
    crossfade(true)     //开启淡入淡出
    crossfade(3000)
    //高斯模糊
    transformations(BlurTransformation(context = applicationContext, radius = 5f, sampling = 5f))
}
```



##### 3.6灰度老照片效果

```kotlin
imageView.load(imageUrl2){
    crossfade(true)     //开启淡入淡出
    crossfade(3000)
    //灰度
    transformations(GrayscaleTransformation())
}
```



##### 3.7加载gif

```kotlin

val imageLoader=ImageLoader.Builder(context = this)
    .componentRegistry{
        if (SDK_INT>28){
            add(ImageDecoderDecoder())
        }else{
            add(GifDecoder())
        }
    }.build()
Coil.setImageLoader(imageLoader)
imageView.load("https://img-blog.csdnimg.cn/c271ed8cf0f541b08e1192d34e512448.gif")
```



##### 3.8检测整个加载的过程

```kotlin
imageView.load(imageUrl2){
    crossfade(true)     //开启淡入淡出
    crossfade(3000)
    //灰度
    transformations(GrayscaleTransformation())
    listener(
        onStart ={ request ->
            Log.d("lpf", "onError 开始加载...")
        },
        onError = { request, throwable ->
            Log.d("lpf", "onError 加载失败...")
        },
        onCancel = { request ->  
            Log.d("lpf", "onCancel 加载重载中...")
        },
        onSuccess = { request, metadata ->
            Log.d("lpf", "onSuccess 加载成功...")
        }

    )
}
```



#### 4.通过自定义Transform实现其他的功能

框架本身提供了：

* GrayscaleTransformation：灰度旧照片效果
* BlurTransformation：高斯模糊效果
* CircleCropTransformation：圆图片效果 
* RoundedCornersTransformation：增加圆角效果

同时还允许用户继承Transformation接口实现自己的功能。

```kotlin
public interface Transformation {
    public abstract fun key(): kotlin.String

    public abstract suspend fun transform(pool: coil.bitmap.BitmapPool, input: android.graphics.Bitmap, size: coil.size.Size): android.graphics.Bitmap
}
```

实现这个接口，并实现key和transform方法就能实现自己的效果

##### 4.1通过自定义Transform实现颜色滤镜效果

```kotlin
class CustomColorFilterTransformation(
    @ColorInt private val color: Int  
) : Transformation {

    override fun key(): String = "${CustomColorFilterTransformation::class.java.name}-$color"

   
    override suspend fun transform(pool: BitmapPool, input: Bitmap, size: Size): Bitmap {
        //获取添加效果图片的配置信息
        val width = input.width  
        val height = input.height  
        val config = input.config  

		//从复用池里边获取最合适的Bitmap图片
        val output = pool.get(width, height, config)
		
		//创建画布和画笔
        val canvas = Canvas(output) 
        val paint = Paint()
        画笔的抗锯齿
        paint.isAntiAlias = true  
        
        //颜色混合
        paint.colorFilter = PorterDuffColorFilter(color, PorterDuff.Mode.SRC_ATOP)
		
		//绘制到图片上
        canvas.drawBitmap(input, 0f, 0f, paint) 

        return output  
    }
}
```

* key()：此方法的返回值是用于计算图片在内存缓存中的唯一 Key 时的辅助参数，所以需要实现该方法，为 Transformation 生成一个可以唯一标识自身的字符串 Key。
* transform()：第一个参数是BitmapPool，我们在实现图形变换的时候往往是需要一个全新的 Bitmap，此时就应该通过 BitmapPool 来获取，尽量复用已有的 Bitmap。
* PorterDuffColorFilter 颜色混合方法参考文档：https://www.twle.cn/l/yufei/android/android-basic-porterduffcolorfilter.html



##### 4.2通过自定义Transform实现水印效果

```kotlin
class CustomWatermarkTransformation(
    private val waterMarkContent: String, 
    @ColorInt private val waterMarkColor: Int,  
    private val waterMarkSize: Float 
    ) : Transformation {

    override fun key(): String = "${CustomWatermarkTransformation::class.java.name}-${watermark}-${textColor}-${textSize}"

    
    override suspend fun transform(pool: BitmapPool, input: Bitmap, size: Size): Bitmap {
        val width = input.width  
        val height = input.height  
        val config = input.config  

  
        val output = pool.get(width, height, config)

        val canvas = Canvas(output) 
        val paint = Paint()  
        paint.isAntiAlias = true 
        paint.textSize = waterMarkSize  
        paint.color = waterMarkColor  
		
        //先绘制目标bitmap
        canvas.drawBitmap(input, 0f, 0f, paint) 

        //绘制水印
        canvas.rotate(40f, width / 2f, height / 2f) 
        val textWidth = paint.measureText(waterMarkContent) 
        canvas.drawText(waterMarkContent, (width - textWidth) / 2f, height / 2f, paint) 
        return output 
    }
}
```

* waterMarkContent：添加水印的文本内容

* waterMarkColor：添加水印的颜色值

* waterMarkSize：添加水印的文本内容的字体大小

