---
title: SDWebImage源码解读与学习（二） # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
SDWebImage源码解读与学习（一）我写的这篇文章已经详细介绍了SDWebImage框架中所有的核心类和具体作用，结尾处也简单示例了一下SDWebImage的使用，相信大家对SDWebImage已经有了一个大体的认识。这次我将带领大家解读一下SDWebImage加载网络图片这个过程。

1、***当我们给UIImageView设置图片URL是，首先调用的是UIView+WebCache这个category中的方法。***
![E49A659E-4846-4B94-8F2A-5F66A12823CC.png](http://upload-images.jianshu.io/upload_images/6644906-74a919d76d926e47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个方法主要干了两件事情。1)取消当前视图正在下载图片的线程2）调用SDWebImageManager的loadImageWithURL方法开始下载图片。当线程下载成功回到到SDWebImageManager的block时，此时会调用
````Objective-c
 [self.imageCache storeImage:transformedImage imageData:(imageWasTransformed ? nil : downloadedData) forKey:key toDisk:cacheOnDisk completion:nil];
````
这个方法将下载的图片缓存到磁盘和内存中

2、***接下来让我分析SDWebImageManager类中loadImageWithURL方法***


![C226D78D-7C3D-49BC-9D85-A47751E182DB.png](http://upload-images.jianshu.io/upload_images/6644906-4846398c5d5c0023.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
该方法中通过如上图SDImageCache去查缓存是否已经有这张图片，如果查到了

![A2245A8E-0D42-4C26-9025-854D9C0413B0.png](http://upload-images.jianshu.io/upload_images/6644906-7d6e794aee939298.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
并不是忙着直接回调，而是判断是否是SDWebImageRefreshCached或者SDWebImageManagerDelegate的方法

  **如果判断条件成立：**
会调用SDWebImageDownloader中downloadImageWithURL启动线程开始下载图片，SDWebImageDownloaderOperation下载成功之后会做一系列的处理。包括根据格式生成矫正过的UIImage、解压缩图片等等

![F7419FF6-36E7-44D6-802B-1ACE0F31ECC7.png](http://upload-images.jianshu.io/upload_images/6644906-a7027f4456efa0bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**如果判断条件不成立:**

![BBAFDDF7-B13B-48DE-9BA9-8553756AD727.png](http://upload-images.jianshu.io/upload_images/6644906-68fa443f8341635a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图则直接调用callCompletionBlockForOperation:completion:image:data:error:cacheType:finished:url这个方法,这个方法最终回调到UIView+WebCache这个类中，此类然后通过判断是UIImageView或UIButton将图片设置到视图上。

### 总结
整个加载过程就是这样，可以看到SDWebImageManager就像一个大脑一样，负责调度各个类完成图片下载、缓存的功能，让各个类各司其职。
