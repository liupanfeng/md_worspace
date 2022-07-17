#### QT 引入三方库



### QT编译流程

qmake 编译Pro生成makefile

make 执行makefile就行

jom 生成界面源码

uic.exe widget.ui -o ui_widget.h

生产信号槽代码

moc.exe wodget.h moc_widget.cpp



预处理

预处理-头文件加载和宏生成cpp

编译-cpp 到.o或者obj

链接 so lib o obj res a

执行 exe dll so



testqmake

testqmake.pro

SOURCES +=main.cpp

CONFIG +=console



qmake -o makefile testqmake.pro

会生成三个makefile文件和debug和release文件夹

进行

jom /f makefile.Debug



```
#指定库名称
TARGET =libdll
#指定库
TEMPLATE=lib
#配置生成静态库
#CONFIG +=staticlib

#定义宏
DEFINES +=LIBDLL_LIB

#配置lib输出路径
DESTDIR="$$PWD/../../lib"
#配置dll动态库的路径
DLLDESTDIR="$$PWD/../../bin"
```

 



```
#debug release版本设置

CONFIG(debug,debug|release){
    TARGET=libdll_d
}else{
    TARGET =libdll
}
```





```
#跨平台设置

!win64{
    message("not win64")
}

win32{
    message(win32)
}

win32|linux{
    message(win32 or linux)
}


macx{
     message(macx)
}

message($$QMAKESPEC)


```



```c++
//argc: 命令行变量数量  argv：命令行变量数组
int main(int argc,char* argv[]){
    
}
```



c++信号槽 



```c++
connect(信号的发送者，发送的信号，信号的接收者，处理函数（槽函数）)
    
s=new SendMessage(this);
r=new ReceiveMessage(this);
   
connect(s,&SendMessage::sendMessage,r,&ReceiveMessage::recevierMessage);
connect(msBtn,&MsButton::clicked,s,&SendMessage::sendMessage);

```





