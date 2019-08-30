---
title: iOS中如何用OCR高效识别手机号 # 这是标题
date: 2019-07-25 09:30:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- OCR
---
# 背景
最近项目中有个需求，就是让识别手机号。其实按照产品的思路就是外购，而且确实已经开始采购。由于采购过程其实也是漫长的，于是乎本人就准备自己研究一下实现手机号识别。

如果用一些第三方的话，比如百度OCR识别是有限制的，而且就是集成SDK而已，没什么可研究的。最终经过寻寻觅觅，找到了我们今天的主角--[Tesseract](https://github.com/tesseract-ocr/tesseract/wiki/Data-Files)。`Tesseract`是一款由HP实验室开发由Google维护的开源OCR（Optical Character Recognition , 光学字符识别）引擎。

# 引入`Tesseract`

工程引入过程真可谓是一波三折，在自己工程引入的时候遇到了挺多的问题。

首先我们通过Github上的开源代码下载`Tesseract`主工程，在主工程中我们可以看到`Tesseract`会被单独编译成一个动态库

## 引入项目
* 将编译好的`TesseractOCR.framework`导入到工程中
* 将`tessdata`文件夹拖到自己的工程中
* 测试demo
```C
G8Tesseract *tesseract = [[G8Tesseract alloc] initWithLanguage:@"eng"];
tesseract.engineMode = G8OCREngineModeTesseractOnly;
tesseract.pageSegmentationMode = G8PageSegmentationModeAutoOSD;
tesseract.image = [UIImage imageNamed:@"image_sample.jpg"];;
[tesseract recognize];
NSString *recognizedText = tesseract.recognizedText;
NSLog(@"%@",recognizedText);
```
其实通过最简单的三部我们已经完成了`TesseractOCR`集成。是不是很简单啊？其实不然，通过以上简单三步我们可能会遇到以下问题。
## 问题1
```C
dyld: Library not loaded: @rpath/TesseractOCR.framework/TesseractOCR
  Referenced from: /var/containers/Bundle/Application/.../testocr.app/testocr
  Reason: image not found
```
这个问题解决方案就是在Xcode->General->Embedded Binaries 点击+,选择`TesseractOCR.framework`即可解决
## 问题2
```
Error opening data file /var/containers/Bundle/Application/.../testocr.app/tessdata/chi_sim.traineddata
Please make sure the TESSDATA_PREFIX environment variable is set to the parent directory of your "tessdata" directory.
```
这个问题就是我在在引入`tessdata`文件夹的方式的时候导致的。
![1564018584898.jpg](https://upload-images.jianshu.io/upload_images/6644906-e1dd5cced1bcff36.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们只要在导入的时候注意选择`Create folder references`就可以了。

通常通过以上步骤，此时我的demo能够正常运行起来了，而且能够识别出原来工程中的图片

## 高效识别手机号
我们项目中的需求就是通过视频流截取一张图片，然后再通过`TesseractOCR`识别。效果图如下：

![1564019370003.jpg](https://upload-images.jianshu.io/upload_images/6644906-f77a89fa4df63991.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过以上代码集成到我们自己的工程发现一个问题就是手机号识别率很低。因为我们用的是`eng`训练库，所以就考虑是不是水土不服，将他换成中文的训练库。

说干就干我们就去[官方](https://github.com/tesseract-ocr/tessdata)下载最新的4.0.0中文训练库`chi_sim.traineddata`。

* 放到`tessdata`文件夹中
* 更改代码,将`eng`改成`chi_sim`
```C
G8Tesseract *tesseract = [[G8Tesseract alloc] initWithLanguage:@"chi_sim"];
```
本以为会大功告成，反而这里就是重头戏!

### 问题3

```C
allow_blob_division
```

在chi_sim.traineddata文件目录下,使用命令行执行：

```c
combine_tessdata -e chi_sim.traineddata chi_sim.config
```

执行完后，在目录下出现chi_sim.config的文件，打开该文件；
在allow_blob_division        F这一行的前面加#，注释掉

```C
即：# allow_blob_division        F  
```  

然后，在执行命令行：
combine_tessdata -o chi_sim.traineddata chi_sim.config

到此在使用 chi_sim.traineddata文件就不会报read_params_file: parameter not found: allow_blob_division

### 问题4

```C
actual_tessdata_num_entries_ <= TESSDATA_NUM_ENTRIES:Error:Assert failed:in file ../../ccutil/tessdatamanager.cpp, line 53
```
这个问题我集合Google和Github上的Issues得出统一结论：`chi_sim.traineddata`版本库不对应，因为我们下载的是4.0.0最新版，所以按照结论我们就换成一个较低的版本，而且确实可以啦！

不过综合对比下来感觉识别率还不是很好，毕竟最新的训练库代表着识别率最高，所以心里隐隐还是不甘心。

***通过问题3不知大家有没有感觉到`chi_sim.traineddata`可以进行分解，然后再次合成呢？***

通过结合[combine_tessdata](https://www.systutorials.com/docs/linux/man/1-combine_tessdata/)有关介绍此命令的文章，我有了一个大胆的想法，将老版本和新版本的的`chi_sim.traineddata`分别解包。然后将两个集合起来重新合成。

我们通过以下两行即可实现

合成
```C
combine_tessdata -o chi_sim.traineddata chi_sim.
```
解压
```C
combine_tessdata -e chi_sim.traineddata chi_sim.
```

* 解压。首先我们通过命令分别解压最新版和旧版的`chi_sim.traineddata`
>通过解压出来的文件，我们对比发现其实新版比旧版少了几个文件。这时候我有个更大胆的想法，我只要我需求的文件，我只要数字识别库就好了，通过这种方式可以极大的减少包的大小，不过通过尝试以失败告终。因为只要我们需要的库的话，数量与代码中的数量又不匹配了
* 替换。将新版包里的文件全部复制到旧版中进行替换
* 合成。通过合成命名合成一个新的`chi_sim.traineddata`

将我们合成的`chi_sim.traineddata`放到工程中就不会再报问题4了。而且识别率很高。(我自己合成的已经分享出来了在文章末尾)
### 优化识别率

结合上步合成的最新训练库，通过多次尝试，以下代码是识别率最高的了。

```C
G8Tesseract *tesseract = [[G8Tesseract alloc] initWithLanguage:@"chi_sim"];
tesseract.engineMode = G8OCREngineModeTesseractOnly;//必须要设置此模式
tesseract.pageSegmentationMode = G8PageSegmentationModeSparseText;//此模式识别率最高
 tesseract.charWhitelist = @"0123456789";//白名单
tesseract.charBlacklist = @"abcdefghigklmnopqrstuvwxyz-_";//黑名单
tesseract.image = newImage;
tesseract.rect = CGRectMake((KSCREEN_WIDTH-210-16)/2, 62-2, 210, 20+4);
tesseract.maximumRecognitionTime = 5;
[tesseract recognize];
```
是不断的使用中，发现一个问题，用户手机与纸的距离是个问题，如果用户离着太远识别率非常低，只有在一个适当的距离能基本很快就能识别出来，而且正确率非常高。

我的解决方案就是通过在扫描界面上增加一个框，这里能够有效控制用户的距离

![](https://upload-images.jianshu.io/upload_images/6644906-01feb1320aa10ae0.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 总结
最终实现了手机号的识别，不过这个识别速度要2到3秒，稍微慢了点，而且那个距离的解决也不是最好的。各位如果有什么好的解决方案可以与我联系

[chi_sim最新合成库](https://github.com/FlyOceanFish/chi_sim.traineddata)
