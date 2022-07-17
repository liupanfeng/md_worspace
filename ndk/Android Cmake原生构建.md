## Android Cmake原生构建

### 1.Cmake工具介绍

**CMake是一个跨平台的构建工具**，可以用简单的语句来描述所有平台的安装（编译过程）。 能够输出各种各样的makefile或者project文件。CMake并不直接构建出最终的软件， 而是产生其他工具的脚本（如makefile），然后再依据这个工具的构建方式使用。

**CMake是一个比make更高级的编译配置工具**，它可以根据不同的平台、不同的编译器， 生成相应的makefile或vcproj项目，从而达到跨平台的目的。

**Android Studio利用CMake生成的是ninja。ninja是一个小型的关注速度的构建系统。** 

CMake其实是一个跨平台的支持产出各种不同的构建脚本的一个工具。



