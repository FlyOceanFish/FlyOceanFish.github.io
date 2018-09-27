---
title: iOS之xib和Storyboard加载原理 # 这是标题
date: 2018-9-27 13:47:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
为了说明加载的过程，我们通过解析`UIViewController`实例化到呈现给我们能看到视图这中间的过程

`UIViewController`也可以没有xib和StoryBoard进行实例化，这里我们分析的是带有xib或StoryBoard的情况

>不管xib还是Storyboard苹果都给给我们提供了上层api用于我们视图和控制器的实例化，分别对应的类是`UINib`和`UIStoryboard`。
我们使用的是最上层的api。

我们带有xib的`UIViewController`实例化的api如下

```Objective-C
[[ViewController alloc] init];
[[ViewController alloc] initWithNibName: bundle:];
```

带有Storyboard的实例化api如下：

```Objective-C
[storyborad instantiateViewControllerWithIdentifier:]
```
其实接下来苹果会调用`UIViewController`的下列方法

```Objective-C
-(instancetype)initWithCoder:(NSCoder *)aDecoder{
    return [super initWithCoder:aDecoder];
}
```

打断点可以看到`aDecoder`这个不为空，这个对象就是`UINibDecoder`的实例，这个类继承于`NSCoder`。通过这个类我们可以应该猜出`UINibDecoder`给我们做了什么。

其实说白了就是帮我们反序列化了。上边的方法其实仅仅帮我们反序列化了UIViewController。这个时候我们拉线的视图都是空。

之后会调用`-(void)awakeFromNib`这个方法告我们`UIViewController`已经成功从xib或Storyboard加载。如果不是通过xib或Storyboard加载的话，这个方法是不会调用的。

接着会调用
```Objective-C
-(void)loadView{
    [super loadView];
}
```
当我们调用super方法的时候，这个时候其实开始真正加载我们xib上的所有视图，并且会调用每个视图的`-(instancetype)initWithCoder:(NSCoder *)aDecoder`方法,这个时候我们通过xib拉线的视图才真正可以使用了;

如果没有xib，super调用的时候会自动帮我们创建一个`UIView`,并且设置给self.view
