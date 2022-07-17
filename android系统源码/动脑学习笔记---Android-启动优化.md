动脑学习笔记---Android-启动优化



1.查看启动时间

筛选display关键字

adb shell am start -W  包名+全类名



am 命令对应系统源码是Am.java

ActivityRecord  l里边有统计启动的时间 reportLaunchTimeLocked



startTime  

开始     给应用分配内存，这段时间称为不可优化时间



reportLaunchTimeLocked

这个方法里边有thisTime



<item name="android:windowIsTranslucent">true</item>  白屏变为透明

<item name="android:windowBackground">@mipmap/test.png</item>  其他页面都会使用这个

弄一个单独的主题，其他的使用统一的主题

在启动页面还需要将主题还原  -》在启动页面调用setTheme（R.style.AppTheme）



App启动原理

App启动包含Application的启动和Activity的启动

adb shell dumpsys activity activities





startActivtySafely

