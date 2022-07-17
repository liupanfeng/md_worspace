### Android OpenGL Camera预览画面渲染

#### 1.OpenGL

OpenGL：全称（Oen Graphics Library）图形绘制语言也是GPU显卡语言，是图形领域的工业标准，是一套跨编程语言、跨平台的、专业的图形编程（软件）接口。它用于二维、三维图像，是一个功能强大，调用方便的底层图形库。

与硬件无关，可以在不同的平台windows、Linux、Mac、Android、IOS之间进行移植。因此支持OpenGL的软件具有很好的移植性，可以得到广泛的应用。

在移动端，使用的是OpenGLES，这个是一个专门针对客户端的精简版本。



#### 2.OpenGLES 支持的版本

OpenGLES 1.0和1.1 --Android1.0和更高的版本支持。

OpenGLES2.0           --Android2.2（API8）和更高的版本支持。

OpenGLES3.0           --Android4.3（API18）和更高的版本支持。

OpenGLES3.1           --Android5.0（API21）和更高的版本支持。

以上是跟Android系统的对应关系



除此之外，还需要由设备制造商提供支持，目前广泛支持2.0的版本，在直接配置2.0的版本就行。

```shell
<uses-feature android:glEsVersion="0x00020000" android"required="true"/>
```



#### 3.GLSurfaceView

GLSurfaceView:这个控件继承SurfaceView，不仅拥有了SurfaceView的所有的功能，还拥有了OpenGL的处理能力。

GLSurfaceView内嵌了surface专门负责OpenGL渲染，管理Surface和EGL。

允许自定义渲染器render，让渲染器在独立的线程里边运作，和UI线程分离。

支持按渲染（on-demand）和连续渲染（continuous）两种模式。

OpenGL是一个跨平台的操作GPU的API，但OpenGL需要本地视窗系统进行交互，这就需要一个中间控制层，EGL就是连接OpenGL ES和本地视图窗口的系统接口，引入EGL就是为了屏蔽不同平台上的区别。



#### 4.OpenGL的绘制流程

三角形是图像领域的最小单元。

* 位置排版，顶点位置确定好（确定顶点位置）
* 根据各个位置排版，细节化网格排版（光栅化）
* 根据各个位置排版，细节的网络排版执行逻辑处理（光栅化逻辑处理）
* 颜色测才准备工作（纹理过滤）
* 片元处理，进行上色处理，绘制成效果画面（纹理填充）
* 输出结果，GPU进行处理成人类能看到的画面。



#### 5.进入实践



笔者这里封装了一个渲染控件`OpenGLView`，继承自`GLSurfaceView`

```java
/**
 * 相当于 自定义控件而已，只不过此自定义控件要显示，
 * Camera预览的画面（OpenGL的处理）
 */
public class OpenGLView extends GLSurfaceView {

    public OpenGLView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }
	
    private void init() {

        /*
        *	设置EGL版本
        * 	2 代表是 OpenGLES 2.0
        * */
        setEGLContextClientVersion(2);

         /*
        * 设置渲染器
        * EGL 开启一个 GLThread.start  run { Renderer.onSurfaceCreated 					 	*	...onSurfaceChanged  onDrawFrame }
        * 如果这三个函数，不让GLThread调用，会崩溃，所以他内部的设计，必须通过GLThread调用来调		* 用三个函数
        * */
        setRenderer(new OpenGlRenderer(this)); 

         /*
        * 设置渲染器模式
        * RENDERMODE_WHEN_DIRTY 按需渲染，有帧数据的时候，才会去渲染（ 效率高，麻烦，后面需要		   * 手动调用一次才行）
        * RENDERMODE_CONTINUOUSLY 每隔16毫秒，读取更新一次，（如果没有显示上一帧）
        * */
        setRenderMode(RENDERMODE_WHEN_DIRTY); 
    }
}
```



`OpenGLRenderer`的实现了GLSurfaceView.Renderer接口

```java
/**
 * 自定义渲染器
 * 渲染器的三个函数 onSurfaceCreated onSurfaceChanged onDrawFrame
 * 有可用的数据时，回调此函数，效率高，后面需要手动调用一次才行
 */
public class OpenGLRenderer implements
        GLSurfaceView.Renderer // 
    , SurfaceTexture.OnFrameAvailableListener 
{

    private final OpenGLView myGLSurfaceView;
    private CameraHelper mCameraHelper;
    private int[] mTextureID;
    private SurfaceTexture mSurfaceTexture;
    private ScreenFilter mScreenFilter;
    /* 矩阵数据，变换矩阵*/
    float[] mtx = new float[16]; 

    public OpenGLRenderer(OpenGLView myGLSurfaceView) {
        this.myGLSurfaceView = myGLSurfaceView;
    }

    /**
     * Surface创建时 回调此函数
     */
    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        mCameraHelper = new CameraHelper((Activity) myGLSurfaceView.getContext(), // 上下文
                Camera.CameraInfo.CAMERA_FACING_FRONT, // 前置摄像头
                800, 480);

        /* 获取纹理ID 可以理解成画布*/
        mTextureID = new int[1];
        /**
         * 1.长度 只有一个 1
         * 2.纹理ID，是一个数组
         * 3.offset:0 使用数组的0下标
         */
        glGenTextures(mTextureID.length, mTextureID, 0);
        /* 实例化纹理了对象*/
        mSurfaceTexture = new SurfaceTexture(mTextureID[0]); 
        /*绑定好此监听 SurfaceTexture.OnFrameAvailableListener*/
        mSurfaceTexture.setOnFrameAvailableListener(this); 

        mScreenFilter = new ScreenFilter(myGLSurfaceView.getContext());
    }

    /**
     * Surface 改变时 回调此函数
     */
    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        mCameraHelper.startPreview(mSurfaceTexture); // 开始预览
        mScreenFilter.onReady(width,  height);
    }

    /**
     * 绘制一帧图像时 回调此函数
     */
    @Override
    public void onDrawFrame(GL10 gl) {
        /* 每次清空之前的 清理成红色的黑板一样*/
        glClearColor(255, 0 ,0, 0); 
        
        /*
        * mask 细节看看此文章：https://blog.csdn.net/z136411501/article/details/83273874
        * GL_COLOR_BUFFER_BIT 颜色缓冲区
        * GL_DEPTH_BUFFER_BIT 深度缓冲区
        * GL_STENCIL_BUFFER_BIT 模型缓冲区
        * */
        glClear(GL_COLOR_BUFFER_BIT);

        /*
        * 绘制摄像头数据
        * 将纹理图像更新为图像流中最新的帧数据 刷新一下
        * */
        mSurfaceTexture.updateTexImage();  

        /*画布，矩阵数据*/
        mSurfaceTexture.getTransformMatrix(mtx);

        mScreenFilter.onDrawFrame(mTextureID[0], mtx);
    }

    /*
    *  有可用的数据时，回调此函数，效率高，后面需要手动调用一次才行
    * */
    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {
        myGLSurfaceView.requestRender(); // setRenderMode(RENDERMODE_WHEN_DIRTY); 配合用
    }
}
```



`ScreenFilter`的实现

```java
/**
 * 显示到 GLSurfaceView 屏幕
 */
public class ScreenFilter {

    /*着色器代码*/
    private  String vertexSourceV ;
    /*着色器程序ID*/
    private final int mProgram; 

    /* 顶点着色器：位置*/
    private final int vPosition; 
    /*顶点着色器：纹理*/
    private final int vCoord;
    /*顶点着色器：矩阵*/
    private final int vMatrix;  

    /*片元着色器：采样器 摄像头数据*/
    private final int vTexture; 

    
    /*顶点坐标 nio的buffer缓存*/
    private FloatBuffer mVertexBuffer;  
    /* 纹理坐标 nio的buffer缓存*/
    private FloatBuffer mTextureBuffer; 
    /*宽度*/
    private int mWidth; 
    /* 高度*/
    private int mHeight;  

    public ScreenFilter(Context context) {
        String vertexSource = readTextFileFromResource(context, R.raw.camera_vertex); // 查找到（顶点着色器）的代码字符串
        String fragmentSource = readTextFileFromResource(context, R.raw.camera_fragment); // 查找到（片元着色器）的代码字符串

        /*
        *配置顶点着色器
        * 
        * */
        /*创建顶点着色器*/
        int vShaderId = glCreateShader(GL_VERTEX_SHADER);
        /*绑定着色器源代码到 着色器（加载着色器的代码）*/
        glShaderSource(vShaderId, vertexSource);
        /*编译着色器代码（编译阶段：编译成功就能拿到顶点着色器ID，编译失败基本上就是着色器代码字符串写错了）*/
        glCompileShader(vShaderId);
        int[] status = new int[1];
        glGetShaderiv(vShaderId, GL_COMPILE_STATUS, status, 0);
        if (status[0] != GL_TRUE) {
            throw new IllegalStateException("顶点着色器配置失败！");
        }

        /*配置片元着色器*/
        /*创建顶点着色器*/
        int fShaderId = glCreateShader(GL_FRAGMENT_SHADER);
        /*绑定着色器源代码到 着色器（加载着色器的代码）*/
        glShaderSource(fShaderId, fragmentSource);
        /*编译着色器代码（编译阶段：编译成功就能拿到顶点着色器ID，编译失败基本上就是着色器代码字符串写错了）*/
        glCompileShader(fShaderId);
        glGetShaderiv(fShaderId, GL_COMPILE_STATUS, status, 0);
        if (status[0] != GL_TRUE) {
            throw new IllegalStateException("片元着色器配置失败！");
        }

        /*配置着色器程序*/
        /*创建一个着色器程序*/
        mProgram = glCreateProgram();
        /*将前面配置的 顶点 和 片元 着色器 附加到新的程序 上*/
        /*顶点*/
        glAttachShader(mProgram, vShaderId);  
        /* 片元*/
        glAttachShader(mProgram, fShaderId); 
        /* 链接着色器 mProgram着色器程序 是我们的成果*/
        glLinkProgram(mProgram); //
        glGetShaderiv(mProgram, GL_LINK_STATUS, status, 0);
        if (status[0] != GL_TRUE) {
            throw new IllegalStateException("着色器程序链接失败！");
        }

        /*释放，删除着色器，因为用不到 顶点着色器/片元着色器了，只需要有着色器程序mProgram即可*/
        glDeleteShader(vShaderId);
        glDeleteShader(fShaderId);


        /*
        * 获取变量的索引值，通过索引来赋值
        * 顶点着色器里面的如下：
        * */
        /*顶点着色器：的索引值*/
        vPosition = glGetAttribLocation(mProgram, "vPosition"); 
        /* 顶点着色器：纹理坐标，采样器采样图片的坐标 的索引值*/
        vCoord = glGetAttribLocation(mProgram, "vCoord"); 
        /*顶点着色器：变换矩阵 的索引值*/
        vMatrix = glGetUniformLocation(mProgram, "vMatrix"); 

        /*片元着色器：采样器*/
        vTexture = glGetUniformLocation(mProgram, "vTexture"); 


        /*
         * NIO Buffer（就是一个缓存，存和取） 着色器语言绑定
         * 顶点坐标缓存（顶点：位置 排版） -- vPosition
         * */
        /*分配内存 坐标个数 * xy坐标数据类型 * float占几字节*/
        mVertexBuffer = ByteBuffer.allocateDirect(4 * 2 * 4) 
                /*使用本地字节序，例如：大端模式，小端模式，这里设置为：跟随OpenGL的变化二变化*/
                .order(ByteOrder.nativeOrder()) 
                .asFloatBuffer();
        mVertexBuffer.clear();  
        /* OpenGL世界坐标*/
        float[] v = {  
                -1.0f, -1.0f,
                1.0f,  -1.0f,
                -1.0f,  1.0f,
                1.0f,  1.0f,
        };
        mVertexBuffer.put(v);

        /*
        * 纹理坐标缓存（纹理：上色 成果） -- vCoord   === 和屏幕挂钩
        * 分配内存 坐标个数 * xy坐标数据类型 * float占几字节
        * */
        mTextureBuffer = ByteBuffer.allocateDirect(4 * 2 * 4) 
                .order(ByteOrder.nativeOrder()) 
                .asFloatBuffer();
        mTextureBuffer.clear(); 
        
        /*屏幕坐标系*/

        /*旋转 180度 就纠正了*/
        float[] t = {  
                1.0f, 0.0f,
                0.0f,  0.0f,
                1.0f,  1.0f,
                0.0f,  1.0f,
        };

        mTextureBuffer.put(t);
    }

    public void onReady(int width, int height) {
        mWidth = width;
        mHeight = height;
    }

    /**
     * 绘制操作
     * @param mTextureID 画布 纹理ID
     * @param mtx 矩阵数据
     */
    public void onDrawFrame(int mTextureID, float[] mtx) {
        /*设置视窗大小，从0开始，这个是合理的*/
        glViewport(0, 0, mWidth, mHeight);
        /*执行着色器程序*/
        glUseProgram(mProgram); // 


        /*顶点坐标赋值  NIO的Buffer 用它就要归零 */
        mVertexBuffer.position(0); 
        /**
         * 传值（把float[]值传递给顶点着色器）把mVertexBuffer传递到vPosition == size:每次两个xy， stride:0 不跳步
         * 1.着色器代码里面的 标记变量 attribute vec4 mPosition;
         * 2.xy 所以是两个
         * 3.不用管
         * 4.跳步 0 不跳步
         */
        glVertexAttribPointer(vPosition, 2, GL_FLOAT, false, 0, mVertexBuffer);
        /*激活*/
        glEnableVertexAttribArray(vPosition);

        /*纹理坐标赋值*/
        mTextureBuffer.position(0);  
        /*传值（把float[]值传递给纹理）
        *把mTexturBuffer传递到vCoord == size:每次两个xy， stride:不跳步*/
        glVertexAttribPointer(vCoord, 2, GL_FLOAT, false, 0, mTextureBuffer);
        /*激活*/
        glEnableVertexAttribArray(vCoord);

        /*变换矩阵 把mtx矩阵数据 传递到 vMatrix*/
        glUniformMatrix4fv(vMatrix, 1, false, mtx, 0);

        /*vTexture*/
        glActiveTexture(GL_TEXTURE0);
        /*绑定纹理ID --- glBindTexture(GL_TEXTURE_2D ,textureId); 
        *如果在片元着色器中的vTexture，不是samplerExternalOES类型，就可以这样写*/
        glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, mTextureID);  
        /*传递参数 给 片元着色器：采样器*/
        glUniform1i(vTexture, 0); 
        /*通知 opengl 绘制 ，从0开始，共四个点绘制*/
        glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    }

    /**
     * 用于读取 GLSL文件中着色器代码  
     * @param context 上下文
     * @param resourceId 传入raw
     * @return GLSL文件中的代码字符串
     */
    public static String readTextFileFromResource(Context context, int resourceId) {
        StringBuilder body = new StringBuilder();
        try {
            InputStream inputStream = context.getResources().openRawResource(resourceId);
            InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
            BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
            String nextLine;
            while ((nextLine = bufferedReader.readLine()) != null) {
                body.append(nextLine);
                body.append('\n');
            }
        } catch (IOException e) {
            throw new RuntimeException("Could not open resource: " + resourceId, e);
        } catch (Resources.NotFoundException nfe) {
            throw new RuntimeException("Resource not found: " + resourceId, nfe);
        }
        return body.toString();
    }
}
```



顶点着色器的实现

```glsl
attribute vec4 vPosition; 
attribute vec4 vCoord;
uniform mat4 vMatrix; 
varying vec2 aCoord; 

void main() {
    //确定好位置排版
    gl_Position = vPosition; 
    aCoord = (vMatrix * vCoord).xy;

}
```



片元着色器

```
// 导入 samplerExternalOES 
#extension GL_OES_EGL_image_external : require

precision mediump float;
uniform samplerExternalOES vTexture;
varying vec2 aCoord;

void main() {
  
       gl_FragColor = texture2D(vTexture, aCoord);
    
}
```



然后在开启预览的时候调用，将surfaceTexture传进去就可以了

```
mCamera.setPreviewTexture(surfaceTexture);
```