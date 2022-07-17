### Android  EGL

##### 1.EGL是什么？

EGL是OpenGL与窗口系统对应的适配层接口



#### 2.EGL的工作流程

* Display 与原生窗口建立链接
  * EGLDisplay eglGetDisplay
  * EGLBoolean eglInitialize
* Surafce 配置和创建surface（窗口和屏幕上的渲染区域）
  * EGLBoolean elgChooseConfig
  * EGLSurface eglCreateWindowSuraface
* Context创建渲染环境（Contenxt上下文）
  * 渲染环境指OpenGL ES的所有项目运行需要的数据结构，如顶点、片段和顶点数据矩阵等
  * eglCreateContext
  * eglMakeCurrent

#### 3.着色器语言GLSL

* 顶点着色器是这对每个顶点执行一次，用于确定顶点的位置，片段着色器是针对每个片段（可以理解为每个像素）执行一次，用于确定每个片段（像素）的颜色。
* GLSL的基本语法和C基本相同
* 不同的是他完美的支持了向量和矩阵
* GLSL提供了大量的内置函数来提供丰富的功能扩展

#### 4.顶点着色器

顶点着色器被使用在传统的基于顶点的操作，例如位移矩阵、计算光照方程、产生贴图坐标等顶点转换操作。







