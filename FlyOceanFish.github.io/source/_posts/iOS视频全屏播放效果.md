---
title: iOS视频全屏播放动画的实现 # 这是标题
date: 2018-05-17 10:04
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 序言
相信大家手机中肯定会装一款看小视频的软件，毕竟现在小视频这么火嘛。不过作为开发人员除了关心视频新闻等信息外，当然也会关注人家具体怎么实现嘛。视频播放的话作者已经封装了一个单独的视频播放器控件[FOFMoviePlayer](https://github.com/FlyOceanFish/FOFMoviePlayer),还没来得及写一篇文章详细介绍，望见谅哈。这次实现的就是视频播放的全屏播放效果。
# 效果
![](https://upload-images.jianshu.io/upload_images/6644906-da610e675350c546.gif?imageMogr2/auto-orient/strip)
# 实现思路
## 进入动画
（动画实现其实很多时候并不是一下子就能想到了，只是一点一点基础上累加实现的，所以大家在实现动画的过程中一开始不要一口吃个胖子，想着一下子完美实现。这个动画作者就是在反复修改，逐渐优化最终实现的）这里作者是通过`UIViewController`的转场动画实现的
### 第一步 大体实现
一开始其实想法很简单就是旋转+缩放。确实这也是根本。
但是有3个问题：
* 旋转位置不对，默认是以试图的中心进行旋转，我们期望以右上角为中心进行旋转
* 缩放不对，我们期望长款正好缩放到屏幕大小
* 旋转之后位置不对，我们期望旋转之后试图平移到屏幕边缘

### 第二部 细节优化
针对以上三个问题，我们的解决方案如下：
* 旋转位置
```Objective-C
snapshot.layer.anchorPoint = CGPointMake(0, 0);
```
`layer`的锚点即`anchorPoint`是试图的中心，我们做任何`layer`层的动画都是以它为中心的。
* 大小缩放到屏幕大小
```Objective-C
CABasicAnimation *animal = [CABasicAnimation animationWithKeyPath:@"transform.scale.x"];
animal.fromValue = @(1);
animal.toValue = @(screenHeight/width);
CABasicAnimation *animal3 = [CABasicAnimation animationWithKeyPath:@"transform.scale.y"];
animal3.fromValue = @(1);
animal3.toValue = @(screenwidth/height);
```
这里我们详细的控制x、y坐标的缩放比例。通过试图的宽高和屏幕的宽高计算所得
* 旋转位置不对
```
CABasicAnimation *animal4 = [CABasicAnimation animationWithKeyPath:@"position"];
animal4.fromValue = @(snapshot.frame.origin);
animal4.toValue = @(CGPointMake(screenwidth, 0));
```
我们通过position动画，让其运到到屏幕边缘，不过大家需要注意的是toValue的x的值是screenwidth

通过以上两步即可实现进入的动画。接下来我们实现返回的动画
## 返回动画
我们实现返回的动画也是基于以上2步来实现的。
如果理解了进入动画的话，返回动画相信你也会写出大部分。不过需要注意的一点就是：
由于在SecondViewController中强制旋转了90度
```Objective-C
self.view.transform = CGAffineTransformRotate(CGAffineTransformIdentity,M_PI_2);
```
所以我们截图的时候，截取的是旋转前的图片。从而我们在做动画之前也要将所截之图旋转一下，在这个基础上再进行之后的动画
```Objective-C
UIView *iv = fromView.subviews[0];
iv = [iv snapshotViewAfterScreenUpdates:NO];
CGAffineTransform transform = CGAffineTransformRotate(CGAffineTransformIdentity, M_PI_2);
iv.transform = transform;
iv.frame = CGRectMake(0, 0, screenwidth, screenHeight);
```
这里我们返回的时候也要在旋转之后的基础上做动画。即UIView的CGAffineTransformRotate是上边的transform上进行的
```
[UIView animateWithDuration:0.3 animations:^{
    CGAffineTransform transform2 = CGAffineTransformRotate(transform, -M_PI_2);
    iv.transform = transform2;
}];
```
# 完整代码
大家可以通过github下载完整代码详细研究，以上部分都是关键核心部分
[FullScreenTransitionAnimal](https://github.com/FlyOceanFish/FullScreenTransitionAnimal)
