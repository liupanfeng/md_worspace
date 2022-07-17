### Android-NDK 动态注册与JNI线程

[TOC]

#### 1.动态注册相比于静态注册的优点

* 被反编译后 安全性高一点
* 在native中的调用，函数名简洁
* 编译后的函数标记 较短一些
* 由于一批方法会统一注册，可以直接使用，所以性能略有提升。



#### 2.JNI_OnLoad方法

这个方法是调用System.loadLibrary这个方法会触发的一个方法，类似于默认构造方法，可以通过重写这个方法实现一些功能，Crash的监听、native方法注册等。

这个方法包含两个参数：

* JavaVM * vm：全局的，相当于App进程的全局成员，JNIEnv可以通过它来得到。
* void * args：这个参数使用很少

#### 3.动态注册的流程

##### 3.1编写需要动态注册的方法

```java
public native int doAction(String name)
```

##### 3.2增加结构体数组

由于动态注册需要，提前写好，存放的是需要动态注册的方法，有多少就写多少。

```c++
static const JNINativeMethod methods[] = {
        {"doAction", "(Ljava/lang/String;)I", (jint *) (doAction)}
};

//这是JNINativeMethod 结构体，系统层定义的
typedef struct {
    const char* name;       
    const char* signature;  
    void*       fnPtr;     
} JNINativeMethod;
```

* name ： 动态注册JNI的函数名 --- Java的动态注册函数
* signature：函数签名 --- Java的动态注册函数签名
* fnPtr：函数指针 -- C++的函数

name牵扯java层的方法，中间是java方法签名，fnPtr牵扯c++方法。

##### 3.3编写JNI_OnLoad方法

```c++
//定义全局变量
JavaVM * javaVM;

jint JNI_OnLoad(JavaVM *vm,void *args){

    ::vm=vm;

    JNIEnv *env;
    //通过JavaVM 得到env
    jint r=vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6);

    if (r!=JNI_OK){
        return -1;
    }

    jclass mainActivityClazz=env->FindClass("com/meishe/ndkcollection/MainActivity");
    //进行注册  第二个参数是一个结构体指针  指针就是地址 数组的名就是首个元素的地址
    //第三发参数是需要注册的方法个数
    r=env->RegisterNatives(mainActivityClazz,methods,
                           sizeof methods/ sizeof(JNINativeMethod));

    if (r!=JNI_OK){
        return -1;
    }else{
        LOGI("动态注册成功\n");
    }

    return JNI_VERSION_1_6;  //一般会使用最新的版本

}
```

注册方法：RegisterNatives(jclass clazz, const JNINativeMethod* methods,
    jint nMethods)参数解析：

* jclass ：字节码就是需要注册方法所在的类对象
* methods：结构体指针，也就是结构体数组的首地址
* 需要动态注册的方法个数

#### 4.JNI中子线程

不管是Android中使用的Thread，还是linux用到的Thread，都是通过pthread_create来创建出来的，如果需要使用这个方法创建子线程。

```c++
#include <pthread.h>


//定义辅助类，用于接受一个jobject对象
class MSContext{
public:
    jobject instance= nullptr;   
};

//声明全局变量，用于close方法，对全局变量的释放
MSContext *msContext;

//声明一个启动方法，java层就不再粘贴了
extern "C"
JNIEXPORT void JNICALL
Java_com_meishe_ndkcollection_MainActivity_naitveThread(JNIEnv *env, jobject thiz) {

    msContext=new MSContext();
    msContext->instance=env->NewGlobalRef(thiz);  //把局部成员 提升为 全局成员

    pthread_t pid;
	//创建一个子线程 
    pthread_create(&pid,nullptr,cpp_thread_run,msContext);

}

//启动一个子线程，需要回调的方法
void *cpp_thread_run(void * args){
    LOGI("C++ Pthread 的异步线程");
    MSContext *msContext= static_cast<MSContext *>(args);

    JNIEnv *env;

    jint r=::vm->AttachCurrentThread((&env), nullptr);
    if (r!=JNI_OK){
        return nullptr;
    }

    jclass activityClazz=env->GetObjectClass(msContext->instance);
    jmethodID update=env->GetMethodID(activityClazz,"runThread","()V");
    env->CallVoidMethod(msContext->instance,update);
	
    //释放JNIEnv等
    ::vm->DetachCurrentThread();

    return nullptr;

}

```

核心方法：int pthread_create(pthread_t* __pthread_ptr, pthread_attr_t const* __attr, void* (*__start_routine)(void*), void*);参数分析：

* __pthread_ptr：线程id
* __attr：参数集合，很少用到，直接传nullptr
*  void* (*__start_routine)(void*)：函数指针，相当于声明一个Thread，回调的run方法 声明的cpp_thread_run这个方法就是按照这个规范写的。
* void*：这个参数就是函数指针的输入的参数，这个是用来进行线程之间通信用的。
