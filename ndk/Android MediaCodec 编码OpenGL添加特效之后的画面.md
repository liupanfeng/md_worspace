### Android MediaCodec 编码OpenGL添加特效之后的画面

使用MediaCodec 进行编码有两个方式：

1.自己实现输入过程交给Input，然后从outByffer去获取压缩之后的数据，然后封包。

2.使用Serface作为输入Buffer，直接从输出Buffer中获取数据。

第二种方式，相对来说会比较简单一些，下面采用第二种方式来处理。

还面临一个问题如何将OpenGL着色器处理之后的数据交给Surface，就需要接触EGL了。