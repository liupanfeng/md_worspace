### Android 性能优化-Apk安装包瘦身

本篇包含：apk安装包内容介绍、CPU指令集的兼容性、AndResGuard的使用。

[TOC]

#### 1.apk安装包内容解析

![image-20220510153046410](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220510153046410.png)

* **lib**： 包含特定于处理器软件层的编译代码。该目录包含了每种平台的子目录，像armeabi，armeabiv7a， arm64-v8a，x86，x86_64，和mips。大多数情况下我们可以只用一种armeabi-v7a，但目前各 大应用市场都要支持64位，所以也会加上arm64-v8a。
* **assets**： 包含应用可以使用AssetManager对象检索的应用资源。
*  **res**： 包含未编译到的资源 resources.arsc,主要有图片资源文件。 
* **META-INF**：包含CERT.SF和 CERT.RSA签名文件以及MANIFEST.MF 清单文件。 
* **resources.arsc**： 包含已编译的资源。该文件包含res/values/ 文件夹所有配置中的XML内容。打包工 具提取此XML内容，将其编译为二进制格式，并将内容归档。此内容包括语言字符串和样式，以及直接 包含在resources.arsc文件中的内容路径 ，例如布局文件和图像。
* **classes.dex**： 包含以Dalvik / ART虚拟机可理解的DEX文件格式编译的类。
*  **AndroidManifest.xml**： 包含核心Android清单文件。该文件列出应用程序的名称，版本，访问权限和 引用的库文件。该文件使用Android的二进制XML格式。

#### 2.CPU指令集的兼容问题

* 只适配armeabi的APP可以跑在armeabi,armeabi-v7a,arm64-v8上
* 只适配armeabi-v7a可以运行在armeabi-v7a和arm64-v8a
* 只适配arm64-v8a 可以运行在arm64-v8a上

CPU指令集具有向下不可兼容的特性，项目中使用那个指令集还需要根据具体的项目情况来决定。











#### 3. AndResGuard的对资源文件的混淆压缩

##### 3.1.AndResGuard介绍

**AndResGuard**是一个帮助你缩小APK大小的工具，他的原理类似Java Proguard，但是只针对资源。他会将原本冗长的资源路径变短，例如将res/drawable/wechat变为r/d/a。

**AndResGuard**不涉及编译过程，只需输入一个apk(无论签名与否，debug版，release版均可，在处理过程中会直接将原签名删除)，可得到一个实现资源混淆后的apk(若在配置文件中输入签名信息，可自动重签名并对齐，得到可直接发布的apk)以及对应资源ID的mapping文件。

##### 3.2.AndResGuard的集成

###### 3.2.1.在外层Build.gradle添加依赖：

```groovy
classpath 'com.tencent.mm:AndResGuard-gradle-plugin:1.2.19'
```

###### 3.2.2.在主工工程build.gradle添加依赖配置：

```groovy
apply from: 'and_res_guard.gradle'
```

###### 3.2.3.and_res_guard.gradle：

```groovy
apply plugin: 'AndResGuard'
andResGuard {

	 // keep住不混淆的资源原有的物理路径 mappingFile = file("./resource_mapping.txt")；
	 //如果混淆全部的话，设置 mappingFile = null
    mappingFile = null
    // 打开这个开关，会keep住所有资源的原始路径，只混淆资源的名字,设置为false就行
    keepRoot = false
    
    // 设置这个值，会把arsc name列混淆成相同的名字，减少string常量池的大小
	// fixedResName = "arg"
    
    // 启用7zip压缩。当你使用v2签名的时候，7zip压缩是无法生效的。
    // use7zip 为true时，useSign必须为true
    // 对于发布于 Google Play 的 APP，建议不要使用 7Zip 压缩，因为这个会导致 Google Play 的优化 Patch 算法失效
    use7zip = true

	// 启用签名，需要配置signConfigs
    useSign = true

   // 保留不被混淆的资源文件，只作用于文件名，不会对路径有影响，支持通配符：? * +
 //  【+】代表1个或多个，【?】代表0个或1个，【*】代表0个或多个。如  "R.id.*",//任意id
    whiteList = [
        // for your icon
        "R.drawable.icon",
        // for fabric
        "R.string.com.crashlytics.*",
        // for google-services
        "R.string.google_app_id",
        "R.string.gcm_defaultSenderId",
        "R.string.default_web_client_id",
        "R.string.ga_trackingId",
        "R.string.firebase_database_url",
        "R.string.google_api_key",
        "R.string.google_crash_reporting_api_key"
    ]

// 打包时是否压缩这类文件，支持通配符：? * +
    compressFilePattern = [
        "*.png",
        "*.jpg",
        "*.jpeg",
        "*.gif",
            //如果不是对APK size有极致的需求，请不要把resources.arsc添加进compressFilePattern
        //"resources.arsc"
    ]
    
    //配置7Zip，只需设置 artifact 或 path；支持同时设置，但此时以 path 的值为优先
    sevenzip {
         artifact = 'com.tencent.mm:SevenZip:1.2.7'
         //path = "/usr/local/bin/7za"  //path指本地安装的7za(7zip命令行工具)
    }

    /**
    * 可选： 如果不设置则会默认覆盖assemble输出的apk
    **/
    //finalApkBackupPath = "${project.rootDir}/final.apk"
    
    /**
    *  可选: 指定v1签名时生成jar文件的摘要算法 默认值为“SHA-1”
    **/
    // digestalg = "SHA-256"
}

```



#### 4.Matrix-ApkChecker 对无用资源的检测

#### 5.so动态加载，借助，腾讯的Tinker的SO加载思想

