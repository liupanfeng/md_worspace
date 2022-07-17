### Android OpenCV 身份证识别实战

#### 1.识别流程

* 手机获取身份证图片
* 计算机图片身份证号码所在的区
  * 需要用到OpenCV，进行图像处理
* OCR文本训练
* 记忆样本移植
* 图像文字识别
  * 使用Tesseract-OCR文字


所以身份证别的关键：找到身份证号所在的区域、获取号码图片



#### 2.图片预处理

* 图片无损压缩

  ```c++
  #define DEFAULT_CARD_WIDTH 640
  #define DEFAULT_CARD_HEIGHT 400
  #define  FIX_CARD_SIZE Size(DEFAULT_CARD_WIDTH,DEFAULT_CARD_HEIGHT)
  
  Mat dst;
  /*图片无损压缩*/
  resize(src_img,src_img,FIX_CARD_SIZE);
  ```

* 图片灰度化，图片的降噪处理：去除噪色提高比对效率
  * 图片压缩，加快图片扫描的速度
  * 图片提取灰度颜色分量，加快图片比对的效率  035911公式
  
  ```c++
  cvtColor(src_img, dst, COLOR_BGR2GRAY);
  ```

* 灰度图片二值化：过滤掉颜色浅的区域，留下关键信息

  ```c++
  threshold(dst, dst, 150, 255, 0);
  ```

* 图像膨胀：膨胀成一个块区域便于轮廓检测

  ```c++
  Mat erodeElement = getStructuringElement(MORPH_RECT, Size(20, 10));
  ```

* 轮廓检测

  ```c++
  findContours(dst, contours, RETR_TREE, CHAIN_APPROX_SIMPLE, Point(0, 0));
  ```

* 图片分割

  ```c++
  rectangle(src_img, finalRect, Scalar(255, 255, 0));
  ```

* 提取身份证核心区域

  ```c++
  Mat dst_img = src_img(finalRect);
  ```



#### 3.Android openCV集成

##### 3.1.将OpenCV头文件放置在main/cpp路径下面

##### 3.2.配置CMakeLists.txt

```shell

cmake_minimum_required(VERSION 3.18.1)


project("msopencv")
#引入头文件
include_directories(${CMAKE_SOURCE_DIR}/include)
#编译源文件
file(GLOB all_file  *.cpp *.c)

add_library(
        msopencv

        SHARED

        ${all_file})

add_library( lib_opencv SHARED IMPORTED)
set_target_properties(lib_opencv PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${CMAKE_ANDROID_ARCH_ABI}/libopencv_java3.so)

find_library( 
        log-lib
        log)


target_link_libraries( # Specifies the target library.
        msopencv
        jnigraphics
        android
        lib_opencv
        ${log-lib})
```



##### 3.3.通过native方法获取图片核心区域

```java
/**
 * 获取图片身份证号核心区域
 */
extern "C"
JNIEXPORT jobject JNICALL
Java_com_meishe_msopencv_ImageProcess_getIdNumber(JNIEnv *env, jclass clazz, jobject src,
                                                  jobject config) {
    Mat src_img;
    Mat dst_img;
    //imshow("src_", src_img);
    /*将bitmap转换为Mat型格式数据*/
    Java_org_opencv_android_Utils_nBitmapToMat2(env, clazz, src, (jlong) &src_img, 0);

    Mat dst;
    /*无损压缩 640*400*/
    resize(src_img, src_img,FIX_IDCARD_SIZE);
    //imshow("dst", src_img);
    /*灰度化*/
    cvtColor(src_img, dst, COLOR_BGR2GRAY);
    //imshow("gray", dst);

    /*二值化*/
    threshold(dst, dst, 150, 255, CV_THRESH_BINARY);
    //imshow("threshold", dst);

    /*膨胀*/
    Mat erodeElement = getStructuringElement(MORPH_RECT, Size(20, 10));
    erode(dst, dst, erodeElement);
    //imshow("erode", dst);

    /*轮廓检测 arraylist*/
    vector< vector<Point> > contours;
    vector<Rect> rects;

    findContours(dst, contours, RETR_TREE, CHAIN_APPROX_SIMPLE, Point(0, 0));

    for (int i = 0; i < contours.size(); i++)
    {
        Rect rect = boundingRect(contours.at(i));
        //rectangle(dst, rect, Scalar(0, 0, 255));  // 在dst 图片上显示 rect 矩形
        if (rect.width > rect.height * 9) {
            rects.push_back(rect);
            rectangle(dst, rect, Scalar(0,255,255));
            dst_img = src_img(rect);
        }

    }
    // imshow("轮廓检测", dst);


    if (rects.size() == 1) {
        Rect rect = rects.at(0);
        dst_img = src_img(rect);
    }else {
        int lowPoint = 0;
        Rect finalRect;
        for (int i = 0; i < rects.size(); i++)
        {
            Rect rect = rects.at(i);
            Point p = rect.tl();
            if (rect.tl().y > lowPoint) {
                lowPoint = rect.tl().y;
                finalRect = rect;
            }
        }
        rectangle(dst, finalRect, Scalar(255, 255, 0));
        //imshow("contours", dst);
        dst_img = src_img(finalRect);
    }

    /*身份证核心区域生成bitmap*/
    jobject  bitmap = createBitmap(env,dst_img,config);


    src_img.release();
    dst_img.release();
    dst.release();

    return  bitmap;

}
```

拿到图片核心区域，返回bigtmap



##### 3.4.通过OCR识别图片上的身份证号信息

###### 3.4.1初始化TessBaseAPI

```java
 mSubscribe = Observable.just(1).observeOn(Schedulers.io()).subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        Log.e("lpf","----doInBackground---");
        mTessBaseAPI = new TessBaseAPI();
        try {
            InputStream is = null;
            is = getAssets().open(mLanguage + ".traineddata");
            File file = new File(PathUtils.getTessDir()+File.separator + mLanguage + ".traineddata");
            if (!file.exists()) {
                file.getParentFile().mkdirs();
                FileOutputStream fos = new FileOutputStream(file);
                byte[] buffer = new byte[2048];
                int len;
                while ((len = is.read(buffer)) != -1) {
                    fos.write(buffer, 0, len);
                }
                fos.close();
            }
            is.close();
            PathUtils.getTessDir();
            mTessBaseAPI.init(PathUtils.getRootDir(), mLanguage);
        } catch (IOException e) {
            e.printStackTrace();
            Log.e("lpf","----copy error:"+e.getMessage());
        }
    }
});
```

###### 3.4.2识别图片上的号码

```java
mTessBaseAPI.setImage(mResultImage);
mTvCardNumberView.setText(mTessBaseAPI.getUTF8Text());
```



总结：这样就通过OpenCV将身份证号识别出来了，其中traineddata数据是训练的结果，文案训练请查看

[身份证号训练教程](https://blog.csdn.net/u014078003/article/details/125351543?spm=1001.2014.3001.5501)

[源码地址](https://github.com/liupanfeng/MSOpenCV.git)

