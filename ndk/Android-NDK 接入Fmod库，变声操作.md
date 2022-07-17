**Android-NDK 接入Fmod库，变声操作**

fmod是音效引擎库，游戏引擎cocos2d、unity3d 等都是默认集成了 fmod 来做音效。

[fmod官网]: https://www.fmod.com/



**1.下载Fmod资源**

可以在官网下载，也可以直接访问百度云盘下载：

链接：https://pan.baidu.com/s/1Ypqlgk8WWacstURNMw2ETw 
提取码：pkom 



**2.接入Fmod**

* 在main路径下新建jniLibs文件夹,增加需要的架构库，根据项目来选择。并讲libfmod.so、libfmodL.so这两个so库拷贝到相应的路径下面。

  * ```groovy
    arm64-v8a：新手机使用的这个架构，微信项目只支持这个，支付宝用的armeabi-v7a。
    ```

  * ```java
    armeabi
    ```

  * ```java
    armeabi-v7a ：游戏类项目，可只选armeabi-v7a，优势具有轻量级的DSP数字信号处理能力。
    
    ```

  * ```java
    x86 ：模拟器以及WindowPhone手机使用的这个架构
    ```

* 在libs路径下面增加fmod.jar包，并在gradle中配置依赖

  * ```java
    implementation fileTree(include: ['*.jar'], dir: 'libs') ：依赖所有libs路径下的jar包。
    ```

  * ```java
    implementation files('libs\\fmod.jar') ：单独依赖此jar包。
    ```

* 在build.gradle中配置

  * ```java
    externalNativeBuild {
        cmake {
            abiFilters "armeabi-v7a"  //指定编译用的CPU架构库
        }
    }
    ```

  * ```java
    ndk {
        abiFilters("armeabi-v7a") //配置最终打包到apk中的包含的CPU架构
    }
    ```

    

* 引入头文件inc，讲inc文件夹全部拷贝到cpp路径下面。

* 配置CMakeLists.txt文件

  * ```cmake
    # 最低支持的CMake版本
    cmake_minimum_required(VERSION 3.18.1)
    ```

  * ```cmake
    include_directories("inc") #导入头文件，因为inc跟CMakeLists.txt文件同级所以直接写inc就行。
    ```

  * ```cmake
    #指定编译的库的名称、类型、以及需要加入的源文件，如果cpp源文件有多个，直接在括号里边添加即可。
    add_library(
            #库名字
            msvoicechange
            # 库类型 动态还是静态
            SHARED
            # 添加的源文件
            native-lib.cpp)
            
    多个cpp源文件的时候，也可以通过统一引用
    	file(GLOB allCPP *.c *.h *.cpp)   
    	${allCPP} 直接添加这个就可以
    ```

  * ```cmake
    #给fmod.so设置环境变量，这样就可以依赖到fmod的动态库了 
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${CMAKE_SOURCE_DIR}/../jniLibs/${CMAKE_ANDROID_ARCH_ABI}")
    ```

  * ```cmake
    
    #这个find_library 其实就是查找 liblog.so 库，并将路径缓存给变量 log-lib，以后直接使用这个变量就行，如果设计多个使用，可以提高效率。 下面是这个so库的路径
    #D:\tools\Android\Sdk\ndk\21.4.7075529\platforms\android-24\arch-arm64\usr\lib\liblog.so
    find_library(
            #变量名称
            log-lib
            #库名   liblog.so   找的是这个库 
            log)
    ```

  * ```
    #进行链接操作
    target_link_libraries( # Specifies the target library.
            msvoicechange
            fmod  #需要将so 链接到msvoicechange 里边  注意这里的名称前面的lib默认会加这里不能写 				#so也是一样
            fmodL
            ${log-lib})
    ```

  

  **3.编写Android上层页面代码**

  加载编译生成的so库，采用静态代码块的方式添加。

  ```java
  companion object {
      init {
          System.loadLibrary("msvoicechange")
      }
  }
  ```

  

  在onCreate方法里边，添加初始化的操作，一般的都会有这样的操作。

  ```java
  override fun onCreate(savedInstanceState: Bundle?) {
      super.onCreate(savedInstanceState)
  
      binding = ActivityMainBinding.inflate(layoutInflater)
      setContentView(binding.root)
  
      FMOD.init(this);
  
  }
  ```

  

  在onDestroy，添加关闭的操作

  ```java
  override fun onDestroy() {
      super.onDestroy()
      FMOD.close();
  }
  ```

  

  定义natvie方法

  ```java
      /**
       * path:音频文件的路径
       * mode：变声的类型：原声、萝莉、大叔等
       */
      external fun voiceChangeNative(mode: Int, path: String)
  ```

  

  **4.编写cpp部分**

  ```cpp
  #include "fmod.hpp"  //有hpp后缀的引用这个，这个是最新版本，加强版本
  #include <unistd.h>
  #include <string>
  
  //增加命名空间
  using namespace std;
  using namespace FMOD;
  
  extern "C"
  JNIEXPORT void JNICALL
  Java_com_meishe_msvoicechange_MainActivity_voiceChangeNative(JNIEnv *env, jobject thiz, jint mode,
                                                               jstring path) {
  
      const char *toast_content = "播放完毕";
  
      //c++只能使用c或者c++的内容  并能用JNI的数据，所以需要转
      //JNI的方法参数，只能使用JNI的类型，所以也需要转
      //得到传进来的音频路径
      const char *audioPath = env->GetStringUTFChars(path, NULL);
  
      //fmod 的音效引擎
      System *system = 0;
      //fmod 的声音
      Sound *sound = 0;
      //fmod 的音轨
      Channel *channel = 0;
  
      //digital signal process  数字信号处理
      DSP *dsp = 0;
  
      //c 的初始化方式xxx(system) 一般都是需要传递指针的，因为需要对地址进行赋值操作，，指针就是地址
      System_Create(&system);
      //初始化
      system->init(32, FMOD_INIT_NORMAL, NULL);
  
      //创建声音
      system->createSound(audioPath, FMOD_DEFAULT, 0, &sound);
      //播放声音
      system->playSound(sound, 0, false, &channel);
  
      switch (mode) {
          case com_meishe_msvoicechange_MainActivity_MODE_NORMAL:
              toast_content = "原声-->播放完毕";
              break;
          case com_meishe_msvoicechange_MainActivity_MODE_LUOLI:
              // 萝莉 音调高
              //创建dsp pitch 音调调节 默认是1  小于1音调低 大于1音调高   0.5-2（合理的调节区间）
              system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT,&dsp);
              //设置pitch音调=2.0
              dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH,2.0f);
              //添加一个音效进去
              channel->addDSP(0,dsp);
              toast_content = "萝莉：播放完毕";
              break;
          case com_meishe_msvoicechange_MainActivity_MODE_DASHU:
              //大叔 音调低
              system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT,&dsp);
  
              dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH,0.7f);
  
              channel->addDSP(0,dsp);
  
              toast_content = "大叔：播放完毕";
              break;
          case com_meishe_msvoicechange_MainActivity_MODE_JINGSONG:
              //惊悚  多个音频轨道 拼接处理
              //音调较低
              system->createDSPByType(FMOD_DSP_TYPE_PITCHSHIFT,&dsp);
              dsp->setParameterFloat(FMOD_DSP_PITCHSHIFT_PITCH,0.7f);
              channel->addDSP(0,dsp);
  
              //echo 回声
              system->createDSPByType(FMOD_DSP_TYPE_ECHO,&dsp);
              dsp->setParameterFloat(FMOD_DSP_ECHO_DELAY,500);
              dsp->setParameterFloat(FMOD_DSP_ECHO_FEEDBACK,35);
              channel->addDSP(1,dsp);
  
              // 颤抖 Tremolo frequency:频率  skew :倾斜
              system->createDSPByType(FMOD_DSP_TYPE_TREMOLO,&dsp);
              dsp->setParameterFloat(FMOD_DSP_TREMOLO_FREQUENCY,0.8);
              dsp->setParameterFloat(FMOD_DSP_TREMOLO_SKEW,0.8);
              channel->addDSP(2,dsp);
  
              toast_content = "惊悚：播放完毕";
              break;
  
          case com_meishe_msvoicechange_MainActivity_MODE_GAOGUAI:
              //搞怪  频率快
              float frequency;
              channel->getFrequency(&frequency);
              channel->setFrequency(frequency*1.5);
              toast_content = "搞怪：播放完毕";
              break;
          case com_meishe_msvoicechange_MainActivity_MODE_KONGLING:
              //空灵
              system->createDSPByType(FMOD_DSP_TYPE_ECHO,&dsp);
              dsp->setParameterFloat(FMOD_DSP_ECHO_DELAY,200);
              //衰减 50 默认  0就没了
              dsp->setParameterFloat(FMOD_DSP_ECHO_FEEDBACK,10);
  
              channel->addDSP(0,dsp);
  
              toast_content = "空灵：播放完毕";
              break;
  
      }
  
      bool isPlay= true;
      while (isPlay){
          channel->isPlaying(&isPlay);
          usleep(1000*1000);  //单位是微秒
      }
  
      //进行资源释放
      sound->release();
      system->close();
      system->release();
      env->ReleaseStringUTFChars(path,audioPath);
  
      //回调java方法，提示用户播放完毕
      jclass _activityClazz = env->GetObjectClass(thiz);
      jmethodID _toastId = env->GetMethodID(_activityClazz, "playerEnd", "(Ljava/lang/String;)V");
      jstring _toastJString = env->NewStringUTF(toast_content);
      env->CallVoidMethod(thiz, _toastId, _toastJString);
  
  }
  ```

  System：fmod 的音效引擎

  Sound：声音

  Channel：音轨

  DSP：数字信号处理

  * pitch:对音调的调节，音调高了一般是萝莉  音调低了就是大叔，默认是1  调节区间（0.5-2）
  * echo:回声，空灵会使用这个
  * Tremolo:这个是颤抖

  

  这样就完成了对Fmod库的依赖，并使用库中的方法做了调音的效果。

  

  

  

  

  

  



