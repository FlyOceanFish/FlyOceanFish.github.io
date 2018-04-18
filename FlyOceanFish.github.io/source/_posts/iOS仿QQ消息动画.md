---
title: iOS仿QQ消息动画 # 这是标题
date: 2018-04-18 18:23:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 动画
---
此次是通过CALayer和CoreGraphics结合实现了QQ消息列表中滑动使数字消失的动画。在滑动的过程中，中间会逐渐变瘦。效果如下图：
![效果](https://upload-images.jianshu.io/upload_images/6644906-5192795aaf4ab435.gif?imageMogr2/auto-orient/strip)
# 实现思路
思路是通过两个圆加一个自己绘制的图形三部分组成，并且是在理想条件下的动画，然后再进行坐标的微调。
>所谓的理想条件如下图1：两个圆大小是一样，同时AB和CD是平行的，AC和BD是两个圆的切线。

![图1](https://upload-images.jianshu.io/upload_images/6644906-a7a5e42490c37ea0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图2](https://upload-images.jianshu.io/upload_images/6644906-ff47a9778653b393.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
根据图1我们算出A、B、C、D、O、P这6个点的坐标（见图2）。O、P两个坐标稍微复杂点。这时候如果我们在AOC这三个点加一条弧线明显是不行的，没有弧度。
所以我们需要调整O、P点的坐标以实现弧度，O、P两个点向里移动，这时候我们在滑动的时候，中间会出现逐渐变瘦的效果。
```Objective-C
CGFloat maxOffx = r1;
CGFloat maxOffy = r1;
CGFloat maxD = 8*r1;//滑动最大距离
CGFloat factor = MIN(maxD, d)/maxD;//这个factor就是需要便宜的百分比
```
这时候我们可以算出O、P偏移之后的坐标
```Objective-C
pointOffO = CGPointMake(pointO.x+maxOffx*factor, pointO.y+maxOffy*factor);
pointOffP = CGPointMake(pointP.x-maxOffx*factor, pointP.y-maxOffy*factor);
```
由于我们滑动过程中会分为四个方向，所以会做如下处理：
```Objective-C
UIBezierPath *ovalPath = [UIBezierPath bezierPath];
[ovalPath moveToPoint:pointA];
CGPoint pointOffO;
CGPoint pointOffP;

if(x1>x2&&y1>y2){
    pointOffO = CGPointMake(pointO.x+maxOffx*factor, pointO.y-maxOffy*factor);
    pointOffP = CGPointMake(pointP.x-maxOffx*factor, pointP.y+maxOffy*factor);
}else if (x1<x2&&y2<y1){
    pointOffO = CGPointMake(pointO.x+maxOffx*factor, pointO.y+maxOffy*factor);
    pointOffP = CGPointMake(pointP.x-maxOffx*factor, pointP.y-maxOffy*factor);
}else if (x1>x2&&y1<y2) {
    pointOffO = CGPointMake(pointO.x-maxOffx*factor, pointO.y-maxOffy*factor);
    pointOffP = CGPointMake(pointP.x+maxOffx*factor, pointP.y+maxOffy*factor);
}else{
    pointOffO = CGPointMake(pointO.x-maxOffx*factor, pointO.y+maxOffy*factor);
    pointOffP = CGPointMake(pointP.x+maxOffx*factor, pointP.y-maxOffy*factor);
}
```
# 完整代码
[Github完整代码](https://github.com/FlyOceanFish/CoreAnimalCALayer)
