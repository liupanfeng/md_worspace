### Android OpenCV 身份证训练 

#### 1.工具安装

下载 jTessBoxEditor-2.2.0 直接解压就好,这个是训练工具

下载地址：https://sourceforge.net/projects/vietocr/files/jTessBoxEditor/



下载这个tesseract-ocr-setup-3.02.02.exe (不要使用太高的版本，可能会不稳定)

下载地址：https://digi.bib.uni-mannheim.de/tesseract/

进行安装,直接下一步就行，没有什么特殊的

安装好之后，查看下 D:\Program Files (x86)\Tesseract-OCR 这个是否配置到path了，如果没有配置到path，进行配置就行



把也要配置到系统变量：

![image-20220618185644892](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618185644892.png)



#### 2.开始训练

提前准备好训练样本图片 .tif格式图片



##### 2.1第一步：将样本图片合并

`java -jar jTessBoxEditor.jar` 启动训练器工具，点击Tools MergeTIFF按钮

![image-20220618182859342](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618182859342.png) 



选择需要训练的所有图片，点击打开会图示让输入保存的文件名称：

![image-20220618183428118](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618183428118.png)



这个文件名必须是如下格式： [lang].[fontname].exp[num]

 lang:语言名(训练生成的示为语言)

 fontname:字体名

 num:序号(无所谓)

 于是可以得到一个命名为zh.song.exp1.tif 的文件



出现这个表述合并成功：

![image-20220618183800812](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618183800812.png)



##### 2.2.生成box文件

```shel
#执行这个命令
tesseract  [lang].[fontname].exp[num].tif [lang].[fontname].exp[num] batch.nochop makebox

[lang].[fontname].exp[num].tif：这个是上一步合并的训练图片

[lang].[fontname].exp[num]：名字与上面的相同，这个是保存box文件的名字

batch.nochop makebox：这个是固定的内容

#最终命令
tesseract zh.song.exp1.tif  zh.song.exp1 batch.nochop makebox
cmd命令行：
Tesseract Open Source OCR Engine v4.0.0.20181030 with Leptonica
Page 1
Page 2
...
Page 18
Page 19

表示成功了，会生成：zh.song.exp1.box文件
```



##### 2.3.校准box文件

继续使用训练工具：选择Box Editor->Open->选择合并的.tif文件

![image-20220618190319257](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618190319257.png)



然后将识别错误的字段修改成正确的值：

![image-20220618190816453](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618190816453.png)

全部改完点击Sava就好

注: 这里必须修改识别错误的字符，否则做出来的traineddata文件也是错的。可以在下面的界面中修改并保存，也可以直接在traineddata文件中修改。



##### 2.4.训练-创建 font_properties

```shell
#在训练的当前目录创建 font_properties 文件
#注意此文件没有后缀.txt
文件内容为:song 0 0 0 0 0

内容解释：
fontname:字体名
italic:斜体       	0/1
bold:黑体       		0/1
fixed:默认字体         0/1
serif:衬线字体         0/1
fraktur:德文黑字体     0/1
```



##### 2.5.训练-产生字符特征文件

```shell
#tesseract lang.song.exp1.tif lang.song.exp1 box.train
#number.song.exp1.tif:合并的图片文件
#number.song.exp1	：跟第一个参数同名
#box.train			：训练命令

#执行命令：
tesseract zh.song.exp1.tif zh.song.exp1 box.train

#执行成功后生成两个字符特征文件：
#zh.song.exp1.tr
#zh.song.exp1.txt
```



##### 2.6 训练-计算字符集

```shell
 #unicharset_extractor [lang].[fontname].exp[num].box
 #生成 unicharset 文件
 
 #执行 计算字符集
 unicharset_extractor zh.song.exp1.box
 
 #执行成功后会生成
 #unicharset 文件
```



##### 2.7.训练-聚集字符特征

```shell
#命令：shapeclustering -F font_properties -U unicharset [lang].[fontname].exp[num].tr
#[可以不运行] 生成 shapetable 文件
		 
#mftraining -F font_properties -U unicharset -O [lang].unicharset [lang].#[fontname].exp[num].tr

#生成 [lang].unicharset、inttemp(图形原型文件)、pffmtable(每个字符所对应的字符特征数文件)、#shapetable(如果没有运行shapeclustering) 文件

# 聚集字符特征
mftraining -F font_properties -U unicharset -O zh.unicharset zh.song.exp1.tr

#执行成功后会生成
#inttemp 文件
#pffmtable 文件
#shapetable 文件


```



##### 2.8.训练-生成字符形状正常化特征文件

```shell
#cntraining [lang].[fontname].exp[num].tr
#生成字符形状正常化特征文件 normproto 文件
cntraining zh.song.exp1.tr
```



##### 2.9.文件重命名

```shell
ren shapetable cn.shapetable
ren normproto cn.normproto
ren inttemp cn.inttemp
ren pffmtable cn.pffmtable
```



##### 2.10.生成tessdata文件

```shell
#运行 combine_tessdata [lang].
#得到 *.traineddata 结果
combine_tessdata zh.

#执行成功后就生成了 zh.traineddata 这个就是最终需要用的
```

最终产物：

![image-20220618204946536](https://gitee.com/weifeng_xixi/images/raw/master/img/image-20220618204946536.png)

#### 3.将一些步骤通过批处理来操作

```shell
#产生字符特征文件 执行命令：
tesseract zh.song.exp1.tif zh.song.exp1 box.train
#执行 计算字符集
unicharset_extractor zh.song.exp1.box
# 聚集字符特征
mftraining -F font_properties -U unicharset -O zh.unicharset zh.song.exp1.tr
#生成字符形状正常化特征文件 normproto 文件
cntraining zh.song.exp1.tr

#重命名
ren shapetable cn.shapetable
ren normproto cn.normproto
ren inttemp cn.inttemp
ren pffmtable cn.pffmtable

#得到 *.traineddata 结果
combine_tessdata zh.
```

将上面的内容保存到train.tat 就可以批量处理

