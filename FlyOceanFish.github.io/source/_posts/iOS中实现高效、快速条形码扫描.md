---
title: iOS中Zbar实现高效、快速条形码扫描 # 这是标题
date: 2020-3-31 09:44:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
## 背景
条形码、二维码是日常生活中用的较多的功能，我们在开发中之前用的都是ZXing、Zbar这两个开源库。不过在iOS7之后，苹果系统自己实现了一套api，终于可以丢弃第三方，用上苹果爸爸自己的功能了。

首先说说ZXing、Zbar这两个开源库。由于ZXing是用java实现的，Zbar是用C语音实现了，所以Zbar的识别效率远远高于ZXing,所以Zbar一般都是开发者的首选。不过iOS7系统出现自己的api后，系统的自然成为了首选。

按照识别速度来排序的话：系统api>Zbar>ZXing。系统的api识别速度非常的快，而且有效距离比其他两个也大。最主要是系统自带的api在识别条形码的时候即使成90度垂直状，仍然可以很快的识别出来。所以疑问来了，那就直接用系统api就行了呗。

话说本人也是这么想的，不过在实际过程中发现一个重大的问题！！有一些条形码居然识别不了！经过分析此条形码的Code是EAN-128Code这是国内自己定义的，所以苹果不支持。但是实际情况线下还挺多这种条形码的，所以最后不得不换成Zbar。

## Zbar脱坑记

Zbar的集成非常简单，通过Cocopods直接导入即可。然后通过以下代码使用
```C
self.readerView = [[ZBarReaderView alloc] init];
self.readerView.frame = CGRectMake(0,0, preView.frame.size.width, preView.frame.size.height);
self.readerView.tracksSymbols = NO;
self.readerView.readerDelegate = self;
self.readerView.torchMode = 0;
self.readerView.allowsPinchZoom = NO;
ZBarImageScanner *scanner = self.readerView.scanner;
[scanner setSymbology:0 config:ZBAR_CFG_ENABLE to:0];
[scanner setSymbology:ZBAR_CODE128 config:ZBAR_CFG_ENABLE to:1];
[scanner setSymbology:ZBAR_CODE93 config:ZBAR_CFG_ENABLE to:1];
[scanner setSymbology:ZBAR_CODE39 config:ZBAR_CFG_ENABLE to:1];
```
很简单，大功告成！
### 问题1
遇到第一个问题则是，发现同一个单号再次扫描反应非常迟钝，而且扫描速度不是非常的快。这个时候点进去`ZBarReaderView`看了一下属性。发现
```C
// this flag still works, but its use is deprecated
@property (nonatomic) BOOL enableCache;

```
初步怀疑跟这个属性有关，所以通过设置此属性为NO,然后再次测试，发现此问题解决。同时扫描速度提速了
## 再次提速
虽然通过设置`enableCache`属性能够加快扫描速度，不过感觉还是有点慢，因为Zbar默认是全屏幕扫，即当条形码出现在预览框就可以识别到。

通过观察属性发现了`scanCrop`，缩小扫描区域
```C
    self.readerView.scanCrop = CGRectMake((_previewH/2-150)/_previewH, 0, 300/_previewH, 1);//中间区域高度为300的扫描区域
```
>这个扫描区域的设置要非常的注意。我们预览框看到的图片与实际图片不一样，它是一个90度放倒的图片。说白了就是我们预览框x、y坐标对应实际照片的y、x坐标；高对应宽，宽对应高。而且这里所有的值都是小于1的。scanCrop默认值是(0,0,1,1)所以会是全屏扫描

不过通过设置下来，近距离扫描确实已经非常快了。但是满足不了我们的要求。得离着20cm左右扫描速度非常快，离着远了识别速度非常的慢，甚至识别不了。

`ZBarReaderView`所有的属性都翻了一个遍，也没有发现相关属性。

最终迫不得已只能去翻看Zbar源码，试试能不能改善了
## 终极优化
在网上转了一圈没搜到相关信息，所以只能研究研究代码。

通过翻看代码发现，其实Zbar可以通俗的讲分成两部分。通过1、通过预览获取图片数据；2、通过C语言核心逻辑去解析
第二步通过确认没有问题，所以问题出在第一步。由于第一步其实有点老生常谈，所以看了代码之后马上发现了问题。即`ZBarReaderViewImpl_Capture`代码的实现

此代码很多年没有维护了，所以获取预览图片的时候，也是按照当时设备最大分辨率设置。随着时间的发展，设备分辨率也提高了非常多

原来代码
```C
if ([session canSetSessionPreset:AVCaptureSessionPreset640x480])
{
    [session setSessionPreset:AVCaptureSessionPreset640x480];
}
else if ([session canSetSessionPreset:AVCaptureSessionPreset352x288])
{
    [self.session setSessionPreset:AVCaptureSessionPreset352x288];
}
```
原来代码支持图片的分辨率最高是640*480非常的小

改后的代码
```C
if ([session canSetSessionPreset:AVCaptureSessionPreset1920x1080]) {
    [session setSessionPreset:AVCaptureSessionPreset1920x1080];
}
if ([session canSetSessionPreset:AVCaptureSessionPreset1280x720])
{
    [session setSessionPreset:AVCaptureSessionPreset1280x720];
}
else if ([session canSetSessionPreset:AVCaptureSessionPreset640x480])
{
    [session setSessionPreset:AVCaptureSessionPreset640x480];
}
else if ([session canSetSessionPreset:AVCaptureSessionPreset352x288])
{
    [self.session setSessionPreset:AVCaptureSessionPreset352x288];
}
```
我们增加了图片的分辨率即可。此时可以高效、快速、远距离的识别到条形码了。

## 总结

解决这个问题也是循序渐进式，通过自己的努力最终解决了问题。当我们在使用第三方的代码库的时候，遇到问题还是要多看看源码，只有这样我们才能了然于胸。遇到问题解决起来也会游刃有余。





