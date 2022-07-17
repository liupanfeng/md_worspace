### Android  定位c++异常

1.不要打任务断点，直接debug，如果崩溃出在c++层调试位置会直接在异常的位置。



2.通过ndk-stack 定位异常

配置环境变量  NDK_HOME=D:\tools\Android\Sdk\ndk-bundle

命令行执行：

adb  logcat  | ndk-stack -sym  F:/workspace/MSPlayer/app/build/intermediates/cmake/debug/obj/armeabi-v7a



复现崩溃路径就能打印出异常的崩溃c++层的堆栈信息。

