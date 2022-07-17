### Android OpenCV人脸模型训练

OpenCV的下载地址：

https://opencv.org/releases/

https://github.com/opencv/opencv/releases?page=1

#### 1.人脸样本数据采集

正样本100个，

使用OpenCV给我们提供的人脸识别库，通过程序识别人脸将图片保存下来作为正样本

```shell
#OpenCV 提供的人脸识别模型
D:/tools/opencv/build/etc/lbpcascades/lbpcascade_frontalface.xml


```

正样本批处理脚本

```shell
@echo off
@for /l %%a in (0,1,99) do (
 @echo lpf/%%a.jpg 1 0 0 24 24>>lpf.data
)
@pause
```

生成lpf.data正样本数据



```shell
opencv_createsamples -info lpf.data -vec lpf.vec -num 100 -w 24 -h 24
```

lpf.vec 就是二级制数据了



由于正负样本的比例最好是1：3 所以负样本300个，随便找，只要不包含人脸就行

```shell
@echo off
@for /l %%a in (0,1,299) do (
 @echo bg/%%a.jpg>>bg.data
)
@pause
```

生成bg.data 待训练的样本数据





```shell
#进行训练
opencv_traincascade -data data -vec lpf.vec -bg bg.data -numPos 100 -numNeg 300 -numStegs 15 -featureType LBP -w 24 -h 24
-data :执行输出目录
-vec ：正样本
-bg ：负样本
-numPos ：每级分类器训练时所用到的正样本数量
-numNeg  ：每级分类器训练时所用到的负样本数量
-numStages：训练分类器的级别，层数越多，分类器的误差就越小，但是检测会变慢
-featureType ：LBP


```



![image-20220629234018928](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220629234018928.png)

出现这个就表示训练成功了，这样的话，表示样本太少，训练的质量不好，成功后会在/data目录下面生成模型数据



opencv 提供了两套接口

java和c++，建议使用c++



建议使用c++

