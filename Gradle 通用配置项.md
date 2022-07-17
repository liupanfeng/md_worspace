### Gradle 通用配置项

Gradle 是Android studio 标配的构建系统，所以必须对它有基本的认识才行。

**共享变量的定义**

Gradle开发中会遇到很多相同的配置，例如不同的module中都要配置compileSdkVersion、buildToolsVersion等变量的值，这些公共的配置称为共享变量。一般情况下，他们的取值都应该保持一致，那么就需要统一管理这些配置。

一般需要在项目的根目录定义一个common_config.gradle配置文件。

```groovy
ext {
    
    //Android 编译版本相关
    android = [
            versionName      : "1.0.0",
            versionCode      : 1,
            compileSdkVersion: 30,
            buildToolsVersion: "30.0.3",
            minSdkVersion    : 16,
            targetSdkVersion : 30
    ]


    dependencies = [
            appcompat       : 'androidx.appcompat:appcompat:1.2.0',
            material        : 'com.google.android.material:material:1.2.0',
            constraintlayout: 'androidx.constraintlayout:constraintlayout:2.1.3',


            //kotlin
            stdlib          : "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version",
            ktx             : 'androidx.core:core-ktx:1.6.0'
    ]

    testDependencies = [
            //test
            androidTestJunit: 'androidx.test.ext:junit:1.1.3',
            testJunit       : 'junit:junit:4.+',
            testEspresso    : 'androidx.test.espresso:espresso-core:3.4.0',
            testng          : 'org.testng:testng:6.9.6'
    ]


    //混淆相关
    minifyEnable = true
    shrinkResEnable = minifyEnable

    //java相关
    javaVersion = 8
    javaMaxHeapSize = '4G'

    //JSK版本兼容
    sourceCompatibility = this.getJavaVersion()
    targetCompatibility = this.getJavaVersion()

    jvmTarget = '1.8'

}

def getJavaVersion() {
    switch (project.ext.javaVersion) {
        case 6:
            return JavaVersion.VERSION_1_6
        case 7:
            return JavaVersion.VERSION_1_7
        case 8:
            return JavaVersion.VERSION_1_8
        case 9:
            return JavaVersion.VERSION_1_9
        default:
            return JavaVersion.VERSION_1_8
    }
}
```



为了项目中所有的module都能引用到，最好是统一在项目根目录的配置文件中进行引用这个配置

```groovy
apply from:"common_config.gradle"
```



下面是通用配置的应用

```groovy
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}

android {
    compileSdkVersion rootProject.android.compileSdkVersion
    buildToolsVersion rootProject.android.buildToolsVersion

    defaultConfig {
        applicationId "com.***.sdkdemoopt"
        minSdkVersion rootProject.android.minSdkVersion
        targetSdkVersion rootProject.android.targetSdkVersion
        versionCode rootProject.android.versionCode
        versionName rootProject.android.versionName

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    //构建配置
    buildTypes {
        release {
            //指定秘钥信息
            signingConfig signingConfigs.release
            //混淆开关
            minifyEnabled rootProject.minifyEnable
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility rootProject.sourceCompatibility
        targetCompatibility rootProject.targetCompatibility
    }

    //产品维度的配置
    productFlavors {
        _360 {}
        tencent {}
        baidu {}
        oppo {}
        vivo {}
        huawei {}
        xiaomi {}
        googleplay {}
    }

    //如果使用到了友盟需要使用这个配置，友盟中后台的信息就会有渠道的概念了
    productFlavors.all {
            //批量修改，类似一个循序遍历
        flavor -> flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
    }

    //签名相关配置
    signingConfigs{
        release{
            storeFile file("***.keystore")
            storePassword "***"
            keyAlias "***"
            keyPassword "***"
        }

        debug{
            storeFile file("***.keystore")
            storePassword "***"
            keyAlias "***"
            keyPassword "***"
        }
    }

    //自定义工程名称
    variant.outputs.all { output ->
        def buildName = "com.***"
        def type = variant.buildType.name
        if (type == "debug") {
            def apkName = 'app-debug'
            outputFileName = new File("../.././../../../build/outputs/apk/debug", apkName + '_' + type + '.apk')
        } else {
            def releaseApkName = buildName + '_' + variant.productFlavors.get(0).name + '_' + type + "_" + versionName + '_' + releaseTime() + '.apk'
            outputFileName = releaseApkName
        }
    }

    //加载aar包需要的配置，否则会出现找不到aar包的错误
    repositories {
        flatDir {
            dirs 'libs'
        }
        
        //加载一个model中引入的aar，，如果所有的module都需要使用这个module，可以配置在跟目录的build.gradle中
//        flatDir {
//            dirs project(':***Model').file('libs')
//        }
    }

    //如果需要生成aar包
    android.libraryVariants.all { variant ->
        variant.outputs.all {
            def fileName = "baseModel"+'_' + releaseTime()+".aar"
            outputFileName = fileName
        }
    }

    kotlinOptions {
        jvmTarget = rootProject.jvmTarget
    }
}


static def releaseTime() {
    return new Date().format("yyyy-MM-dd--HH-mm-ss", TimeZone.getTimeZone("GMT+8"))
}

dependencies {
    //自动引入libs下面的jar包
    implementation fileTree(includes: ['*.jar'],dir: 'libs')

    implementation rootProject.ext.dependencies.constraintlayout
    implementation rootProject.ext.dependencies.appcompat
    implementation rootProject.ext.dependencies.material

    //依赖一个model
    implementation project(":libBase")

    //引入一个aar
    implementation(name:'libEngine',ext:'aar')

    //kotlin
    implementation rootProject.ext.dependencies.stdlib
    implementation rootProject.ext.dependencies.ktx

    //test
    testImplementation rootProject.testDependencies.testJunit
    androidTestImplementation rootProject.testDependencies.androidTestJunit
    androidTestImplementation rootProject.testDependencies.testEspresso
}
```

我把常用的配置全部放在了里边，包含

* 签名密钥

* 混淆开关

* 自定义apk安装包名称

* 自定义生成aar包名称

* 产品维度，以及包含友盟的配置

* 依赖一个aar包

* 依赖一个子model

  其他配置会持续更新，亲测可用……

  

  

  