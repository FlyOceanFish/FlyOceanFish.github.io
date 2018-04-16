---
title: iOS中CALayer和CoreAnimal以例说教 # 这是标题
date: 2018-04-13 10:00:00
updated: 2018-04-16 11:11:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- CoreAnimal
---
UIKit中我们使用的UIView、UILabel等控件其实显示的载体都是CALayer及其子类，比如CATextLayer、CAScrollLayer等。我们在做Layer层动画的时候则使用的CAShapeLayer比较多。通过CoreAnimal中的CABasicAnimation结合path来实现。

之前都是通过长篇幅的介绍一些理论知识，这次通过一个个的小案例，由简到繁一步步介绍。
# 画正方形、圆形、半圆形等
```Objective-C
//    UIBezierPath *path = [UIBezierPath bezierPathWithRect:CGRectMake(50, 50, 100, 100)];
//    UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(50, 50, 100, 120)];//当宽高一样的时候就是圆,否则就是椭圆
//    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(50, 50, 100, 120) cornerRadius:90];//首先是画一个矩形，然后再设置四个角的角度
UIBezierPath *path = [UIBezierPath bezierPathWithArcCenter:CGPointMake(100, 100) radius:80 startAngle:-M_PI_2 endAngle:M_PI/2 clockwise:YES];//以point为中心，radius的值为半径画弧。此弧的开始角度为startAngle,结束角度为endAngle，这两个数值是弧度，不是角度。clockwise是画此弧是按照顺时针还是逆时针
CAShapeLayer *pointLayer = [CAShapeLayer layer];
pointLayer.path = path.CGPath;
pointLayer.strokeColor = [[UIColor redColor] CGColor];//线条颜色
pointLayer.lineWidth = 10;
//    kCALineCapButt 直角
//    kCALineCapRound 圆角
//    kCALineCapSquare 与kCGLineCapButt是一样的，但是比kCGLineCapButt长一点
pointLayer.lineCap = kCALineCapRound;//当线条是单独的没有与其他线连接的时候，此时线头的形状
[self.view.layer addSublayer:pointLayer];
```

# strokeStart和strokeEnd属性案例
```Objective-C
UIBezierPath *path = [UIBezierPath bezierPathWithArcCenter:CGPointMake(100, 200) radius:80 startAngle:-M_PI_2 endAngle:M_PI*3/2 clockwise:YES];
CAShapeLayer *layer = [CAShapeLayer layer];
layer.lineWidth = 10;
layer.path = path.CGPath;
layer.strokeColor = [UIColor purpleColor].CGColor;
layer.fillColor = [UIColor orangeColor].CGColor;
layer.strokeStart = 0.5;
layer.strokeEnd = 0.8;
[self.view.layer addSublayer:layer];
```
![IMG_0160.jpg](https://upload-images.jianshu.io/upload_images/6644906-4eef721bb65bc574.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
strokeStart和strokeEnd就是笔的开始和结束，值的范围是0到1。
# 渐变层的使用
```Objective-C
CAGradientLayer *layer = [CAGradientLayer layer];
layer.frame = CGRectMake(100, 300, 20, 100);
UIColor *middleColor = [UIColor colorWithWhite:255/255 alpha:0.8];
NSArray *colors = @[
                    (__bridge id)[UIColor redColor].CGColor,
                    (__bridge id)middleColor.CGColor,
                    (__bridge id)[UIColor redColor].CGColor
                    ];
layer.colors = colors;
[self.view.layer addSublayer:layer];
layer.startPoint = CGPointMake(0, 0);
layer.endPoint = CGPointMake(1, 0);
[self.view.layer addSublayer:layer];
```
![IMG_0161.jpg](https://upload-images.jianshu.io/upload_images/6644906-dbe55459250ddc87.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* frame 渐变层是不支持路径的，是通过frame属性来设置大小的，只有长方形和正方形。
* colors 设置需要渐变的颜色
* startPoint 设置渐变的开始位置，x、y值的范围是0到1
* endPoint 设置渐变结束的位置，x、y值的范围是0到1

假设我们想通过路径渐变应该怎么办呢？比如一个半圆弧的渐变（可以思考一下怎么实现，下边会有实现的）

# 遮罩层
以下代码是首先画了一个正方形，填充颜色是红色，然后画了一个圆形。圆形是正方形的遮罩层
如果大家了解ps的话，其实遮罩层就是ps中的蒙版的概念。
遮罩层不会显示出来，<font color='red'>遮罩层有颜色的区域才会将被遮罩层显示出来，没有颜色的不会显示出来</font>
```Objective-C
UIBezierPath *maskPath = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(0, 0, 100, 100) cornerRadius:50];//圆形
CAShapeLayer *maskLayer = [CAShapeLayer layer];
maskLayer.frame = CGRectMake(0, 0, 100, 100);
maskLayer.path = maskPath.CGPath;

UIBezierPath *path = [UIBezierPath bezierPathWithRect:CGRectMake(0, 0, 100, 100)];//正方形
CAShapeLayer *layer = [CAShapeLayer layer];
layer.frame = CGRectMake(50, 50, 100, 100);
layer.path = path.CGPath;
layer.fillColor = [UIColor redColor].CGColor;
layer.mask = maskLayer;//设置遮罩层
[self.view.layer addSublayer:layer];
```
![IMG_0162.jpg](https://upload-images.jianshu.io/upload_images/6644906-46ea3a49f2da89b8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上边的问题不知道大家有没有思路了呢？
# fillRule中kCAFillRuleEvenOdd属性
```
UIBezierPath *path1 = [UIBezierPath bezierPathWithRect:CGRectMake(0, 0, 100, 100)];//正方形
UIBezierPath *path2 = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(0, 0, 100, 100) cornerRadius:50];//圆形
[path1 appendPath:path2];//结合两个路径
CAShapeLayer *shapLayer = [CAShapeLayer layer];
shapLayer.frame = CGRectMake(50, 50, 100, 100);
shapLayer.fillColor = [UIColor blackColor].CGColor;
shapLayer.fillRule = kCAFillRuleEvenOdd;
shapLayer.path = path1.CGPath;
shapLayer.lineWidth = 0.5;


[self.view.layer addSublayer:shapLayer];

CAShapeLayer *layer = [CAShapeLayer layer];
layer.frame = CGRectMake(50, 50, 100, 100);
layer.fillColor = [UIColor redColor].CGColor;
layer.path= path1.CGPath;
[self.view.layer insertSublayer:layer below:shapLayer];//设置一个背景，这样看的效果更好一些
```
![IMG_0163.jpg](https://upload-images.jianshu.io/upload_images/6644906-7638fd667f8e976c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
不知道大家看效果图有没有明白这个属性的作用呢？其实就是对于两个path有重叠部分，取其非交集。比如一个矩形中有一个圆，则填充颜色的区域是这两个的非交集部分。比如我们画两个同心圆，一大一小，则能够实现画出一个扇形。
# 综合以上属性，比较复杂的效果
```
UIBezierPath *path = [UIBezierPath bezierPathWithArcCenter:CGPointMake(100, 100) radius:80 startAngle:-M_PI_2 endAngle:M_PI/2 clockwise:YES];//以point为中心，radius的值为半径画弧。此弧的开始角度为startAngle,结束角度为endAngle，这两个数值是弧度，不是角度。clockwise是画此弧是按照顺时针还是逆时针
   CAShapeLayer *pointLayer = [CAShapeLayer layer];
   pointLayer.path = path.CGPath;
   pointLayer.strokeColor = [[UIColor blackColor] CGColor];//线条颜色


   //    kCAFillModeRemoved 这个是默认值,也就是说当动画开始前和动画结束后,动画对layer都没有影响,动画结束后,layer会恢复到之前的状态（可以理解为动画执行完成后移除）
   //    kCAFillModeForwards 当动画结束后,layer会一直保持着动画最后的状态与CABasicAnimation的removedOnCompletion结合使用
   //    kCAFillModeBackwards 当在动画开始前,你只要把layer加入到一个动画中,layer便立即进入动画的初始状态并等待动画开始.你可以这样设定测试代码,延迟3秒让动画开始,只要动画被加入了layer,layer便处于动画初始状态
   //    pointLayer.fillMode = kCAFillModeBackwards;//是针对动画的，决定当前对象在非active时间段的行为.比如动画开始之前,动画结束之后的状态


   pointLayer.fillRule = kCAFillRuleEvenOdd;//对于两个path有重叠部分，取其非交集。比如一个矩形中有一个圆，则填充颜色的区域是这两个的非交集部分


   //    kCALineCapButt 直角
   //    kCALineCapRound 圆角
   //    kCALineCapSquare 与kCGLineCapButt是一样的，但是比kCGLineCapButt长一点
   pointLayer.lineCap = kCALineCapRound;//当线条是单独的没有与其他线连接的时候，此时线头的形状

   //    kCALineJoinMiter 正常连接
   //    kCALineJoinRound 圆角
   //    kCALineJoinBevel 成斜角，是一条斜线
   //    pointLayer.lineJoin = kCALineJoinRound;//当线条与其他线有连接的时候，连接处最外侧线的显示方式
   pointLayer.fillColor = [UIColor clearColor].CGColor;//填充颜色。这里因为是遮罩层，所以设置为无颜色，则这部分不会显示出来
   pointLayer.lineWidth = 10;


   CASpringAnimation *anim2 = [CASpringAnimation animationWithKeyPath:@"strokeEnd"];//9.0之后才有的阻尼回弹动画
   anim2.fromValue = @0;
   anim2.toValue = @1;
   anim2.duration = 2;
   anim2.timingFunction =
   [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
   anim2.damping = 8;//阻尼系数，值越大停止的越快
   anim2.stiffness =100;//刚度系数(劲度系数/弹性系数)，刚度系数越大，形变产生的力就越大，运动越快
   anim2.mass = 2;//质量，影响图层运动时的弹簧惯性，质量越大，弹簧拉伸和压缩的幅度越大

   //initialVelocity:
   //    初始速率，动画视图的初始速度大小
   //    速率为正数时，速度方向与运动方向一致，速率为负数时，速度方向与运动方向相反

   CAGradientLayer *gradientLayer = [CAGradientLayer layer];
   gradientLayer.frame =CGRectMake(10, 10, 190, 190);
   UIColor *middleColor = [UIColor blueColor];
   NSArray *colors = @[
                       (__bridge id)[UIColor redColor].CGColor,
                       (__bridge id)middleColor.CGColor,
                       ];
   gradientLayer.colors = colors;
   gradientLayer.startPoint =  CGPointMake(0.5, 0.5);
   gradientLayer.endPoint = CGPointMake(0.5, 1);
   gradientLayer.mask = pointLayer;//设置渐变层的遮罩层为我们前边画的半弧，则只会显示板湖那部分区域

   [self.view.layer addSublayer:gradientLayer];
   [pointLayer addAnimation:anim2 forKey:@"animal"];
```
![动画.gif](https://upload-images.jianshu.io/upload_images/6644906-13134c2bf6b9483d.gif?imageMogr2/auto-orient/strip)
# 关键帧动画
CAKeyframeAnimation在这里是实现了一个小圆球沿着一个半圆路径一直旋转运动的动画。（圆心在路径上）
```
UIBezierPath *circle = [UIBezierPath bezierPathWithRoundedRect:CGRectMake(0, 0, 10, 10) cornerRadius:5];
CAShapeLayer *circleLayer = [CAShapeLayer layer];
circleLayer.frame = CGRectMake(0, 0, 10, 10);
circleLayer.fillColor = [UIColor orangeColor].CGColor;
circleLayer.path = circle.CGPath;
[self.view.layer addSublayer:circleLayer];

CAKeyframeAnimation * theAnimation;

// Create the animation object, specifying the position property as the key path.
theAnimation=[CAKeyframeAnimation animationWithKeyPath:@"position"];
theAnimation.path=thePath;
theAnimation.duration=5.0;
theAnimation.repeatCount = 100;
theAnimation.fillMode = kCAFillModeForwards;
[circleLayer addAnimation:theAnimation forKey:@"position"];

CAShapeLayer *animalTrack = [CAShapeLayer layer];
animalTrack.path = thePath;
animalTrack.strokeColor = [UIColor redColor].CGColor;
animalTrack.fillColor = [UIColor clearColor].CGColor;
animalTrack.lineWidth = 1;
[self.view.layer addSublayer:animalTrack];
```
这里一开始有个问题，就是不是圆心沿着路径运动。然后我又换成了UIView试了一下可以的。最后研究主要是circleLayer我们仅仅设置了path,但是frame是没有的，设置了frame之后圆心开始沿着路径运动。
![帧动画.gif](https://upload-images.jianshu.io/upload_images/6644906-1071184cf602e560.gif?imageMogr2/auto-orient/strip)

# 源码
[Github源码](https://github.com/FlyOceanFish/CoreAnimalCALayer)
