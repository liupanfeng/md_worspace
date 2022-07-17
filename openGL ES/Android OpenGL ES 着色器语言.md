# Android OpenGL ES 着色器语言

着色器是OpenGL ES3.0 API的一个基础核心概念，每一个OpenGL ES 程序都需要一个顶点着色器和一个片段着色器，以渲染有意义的图片。

#### 1.OpengGL ES 着色器语言

![image-20220710234304651](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220710234304651.png)

上图就是顶点着色器到片段着色器的一个示意图。



#### 2.常见着色器语言 数据类型

| 类型               | 描述                                            |
| ------------------ | ----------------------------------------------- |
| float              | 浮点型                                          |
| vec2               | 含有两个浮点类型的向量                          |
| vec4               | 含有四个浮点类型的向量                          |
| sampler2D          | 2D纹理采样器（代表一个纹理，使用uniform来修饰） |
| mat2               | 一个2*2的矩阵                                   |
| mat4               | 一个4*4的矩阵                                   |
| samplerExternalOES | android中使用的采样器                           |

在Android中不能直接使用sampler2D类型采样器，需要使用samplerExternalOES采样器

使用samplerExternalOES采样器需要 #extension GL_OES_EGL_image_external : require

#### 3.常用修饰符

| 类型      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| attribute | 属性变量，只能用于顶点着色器中，一般用该变量表示一些顶点数据，如：顶点坐标，纹理坐标 |
| uniform   | 一致变量，在着色器执行期间一致变量的值是不变的，与const常量不同，这个值在编译期间是位置的，是着色器外部初始化的。 |
| varying   | 易变变量，从顶点着色器传递到片断着色器的数据变量             |

#### 4.常用限定符

| 限定符 | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| in     | 这个限定符指定参数按值传送，函数不能更改                     |
| out    | 这个限定符表示该变量的值不能传入函数，但是在函数返回时将会被修改 |
| inout  | 这个限定符规定变量按照引用传入函数，如果该值被修改，它将在函数退出后变化 |

#### 5.常见的内置函数

texture2D(采样器，坐标) 采样指定位置的纹理。

dot 计算两个向量的点积

pow 计算标量的幂次



#### 6.高中低精度

```glsl
//低精度 8位
precision lowp float
//中精度 16位
precision mediump float
//高精度 32位
precision highp float    
```



#### 7.顶点着色器和片段着色器实例

##### 7.1.顶点着色器

```glsl
/*顶点着色器 位置规划的*/
//顶点坐标，相当于：相机的四个点位置排版
attribute vec4 vPosition; 
//纹理坐标，用来图形上色的
attribute vec4 vCoord; 
//变换矩阵，4*4的格式的 
uniform mat4 vMatrix; 
//把这个最终的计算成果，给片元着色器 【不需要Java传递，他是计算出来的】
varying vec2 aCoord; 

void main() {
    //确定好位置排版   gl_Position OpenGL着色器语言内置的变量
    gl_Position = vPosition; 

    //将矩阵和顶点进行对应 取xy两个分量数据 传递给片段着色器
    aCoord = (vMatrix * vCoord).xy;

}
```

##### 7.2.片段着色器

```
/*片元着色器  上色的*/
/*导入 samplerExternalOES */
#extension GL_OES_EGL_image_external : require

/*float 数据的精度 （precision lowp = 低精度） 8位
 *（precision mediump = 中精度）   16位
 *（precision highp = 高精度）    32 位
 */
precision mediump float;

/*
* 根据上面的数据的精度，写下面的 采样器 相机的数据
* 由于是 安卓的相机，就不能用他  sampler2D == GL_TEXTURE_2D
* uniform sampler2D vTexture;
* samplerExternalOES才能采样相机的数据 == GLES11Ext.GL_TEXTURE_EXTERNAL_OES
*/
uniform samplerExternalOES vTexture;
/*
*  把这个最终的计算成果，给片段着色器，拿到最终的成果才能上色
*/
varying vec2 aCoord;

void main() {
    /*
    * texture2D (采样器, 坐标)   opengles 内置函数
    * gl_FragColor OpenGL着色器语言内置的变量
    * vTexture : 是采样器  aCoord是经过处理的纹理坐标
    */
     gl_FragColor = texture2D(vTexture, aCoord);

    // 305911公式：黑白电视效果，其实原理就是提取出Y分量
//    vec4 rgba =texture2D(vTexture, aCoord);
//    float gray = (0.30 * rgba.r   + 0.59 * rgba.g + 0.11* rgba.b); // 其实原理就是提取出Y分量 ,就是黑白电视
//    gl_FragColor = vec4(gray, gray, gray, 1.0);

    /*底片效果  两次上色，就恢复了*/
//    vec4 rgba = texture2D(vTexture, aCoord);  // rgba
//    gl_FragColor = vec4(1.-rgba.r, 1.-rgba.g, 1.-rgba.b, rgba.a);
}
```