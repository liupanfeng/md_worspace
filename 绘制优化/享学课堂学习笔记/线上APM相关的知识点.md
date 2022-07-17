### 线上APM相关的知识点

**技术点**

Bytecode、Hook（PLT hook）、webview的性能检测（JS注入 js）、Gradle插件、ASM、javapoet

**java实现的功能**

* CPU的指标
* 内存的指标
* FPS的指标
* ANR
* 卡顿
* GC/OOM
* 网络检测 （拿到http的上行、下行数据量）Hook
* 功耗
* 远程下发 日志回捞 （Push） 支持远程shell动态的下发



**APM**

1.配置（注解+json）

String[] params default{};

[{

"clazz":"",

"name":"demo",

"params":[

​	"ABC","123"

]

}]



2.数据的保存

App启动、结束、页面

LifecleCallbacks

3.Crash

Thread

4.CPU GC 电量

/proc/stat   /proc/pid/stat   BatteryMonoitor

5.ANR FPS  Choregrapher.FrameCallback

文件检测 /data/anr/trace.txt

