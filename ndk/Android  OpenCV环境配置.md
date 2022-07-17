### Android  OpenCV环境配置

#### 1.需要用到的资源

```shell
#下载：cmake
我使用的版本是：cmake-3.16.2-win64-x64，下载之后进行安装就好

#下载opencv  直接去github下载就可以 (https://github.com/opencv/opencv/releases?page=2)
我使用的版本是：opencv-4.2.0-vc14_vc15.exe，这个双击之后会生成一个文件夹，包含build和source


#因为要集成到android，openCV还要下载一个android版本
我使用的版本是：opencv-4.2.0-android-sdk.zip


#下载mingw
我使用的版本是:mingw-x86_64-8.1.0-release-posix-seh-rt_v6-rev0.7z,直接解压就行，将bin目录配置到环境变量

```



#### 2.配置阶段

#执行cmake gui 
配置cmake源码路径  D:/tools/opencv/sources
配置编译目标生成的路径  D:/tools/opencv/minGW_Build

![image-20220618111835764](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618111835764.png)



#配置编译环境
选择MinGW MakeFile

![image-20220618111723523](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618111723523.png)



#配置minGW gcc和g++

![image-20220618111950836](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618111950836.png)



#执行configure 进行配置就行

在过程当中下载FFmpeg会出现问题：

D:\tools\opencv\sources\.cache\ffmpeg 在这个路径下查看

55c0bc8ad27db00116fabf06508de196-opencv_videoio_ffmpeg_64.dll

5de6044cad9398549e57bc46fc13908d-opencv_videoio_ffmpeg.dll

上面两个文件出现问题会0kb



处理版本：

将 D:\tools\opencv\build\bin 路径下面的

opencv_videoio_ffmpeg420_64.dll

opencv_videoio_ffmpeg420.dll

拷贝到D:\tools\opencv\sources\.cache\ffmpeg 进行重命名就行



#### 3.进行编译

#在C:\Windows\System32\cmd.exe 下打开cmd 执行
`minGW32-make -j 4`   进行编译

出现下图，就编译成功了

![image-20220618115828505](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-202206181158285.png)



`minGW32-make install`  进行安装，出现下图就表示安装成功了

![image-20220618120600226](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220618120600226.png)



安装成功后会生成一个install路径

将 D:\tools\opencv\minGW_Build\install\x64\mingw\bin 路径配置到环境变量即可。



#### 4.使用Clion 进行配置

新建一个C++工程，配置Toolchains

![image-20220618134534684](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220618134534684.png)



检测OpenCV配置的环境

```c++
#include <iostream>
#include <opencv2/opencv.hpp>

using namespace cv;
int main() {
    Mat src=imread("D:\\pic.png");
    imshow("aaa",src);
    waitKey(0);
    return 0;
}
```



配置CmakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.17)
project(FaceTrain)

set(CMAKE_CXX_STANDARD 14)

add_executable(FaceTrain main.cpp)

#修改为自己的路径
set(OpenCV_DIR D:/tools/opencv/minGW_Build)
find_package(OpenCV REQUIRED)

include_directories(${OpenCV_INCLUDE_DIRS})

message(STATUS "${OpenCV_INCLUDE_DIRS}")
message(STATUS "${OpenCV_LIBS}")


target_link_libraries( FaceTrain ${OpenCV_LIBS} )
```



执行程序，如果能显示出图片，说明环境配置成功了。