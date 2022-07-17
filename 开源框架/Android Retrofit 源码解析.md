### Android Retrofit 源码解析



#### 1.Retrofit概述

AOP面向切面编程应用的案例：

* leakCanary：在某一个点进行内存泄漏的检测

* BlockCanary：在某一个切面对卡顿进行监控
* Matrix：卡顿的监控
* OkHttp 拦截器也是面向切面编程

Retrofit核心思想就是面向切面编程：Retrofit本身并不具备网络的功能，是对OkHttp功能的封装，需要将各种各样的请求转换成OKHttp请求，那么就需要一个切面进行统一的转换。

Retrofit框架：轻量级，代码量不大，但是功能强大。非常巧妙的使用了多种设计模式，非常值得研究。



#### Retrofit 解决了那些问题



OkHttp：非常优秀的一个网络请求库，OKHttp是基于Http协议封装的一套请求客户端，偏向于网络请求。OkHttp主要负责socket部分的优化，比如多路复用，buffer缓存，数据压缩等。

OkHttp存在的问题：



但是不具备好用的特点

PkHttp存在的问题



