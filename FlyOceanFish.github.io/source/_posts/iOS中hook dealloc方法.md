---
title: 能够hook住dealloc方法吗？ # 这是标题
date: 2017-11-20 09:10:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
一个朋友群里讨论能够hook住dealloc方法吗？然后当时我就想为什么hook不住吗？所以我就在群里问为什么呢？奈何他也不知道。所以剩下的只能是我自己去尝试一下了。

大家hook方法的时候第一步肯定是首先获取到方法的实现。用到如下代码:

```Objective-C
Method repMethod = class_getInstanceMethod([self class], @selector(replaceDelloc));
```
奈何我用上述方法写的时候Xcode编译器会报编译错误

>ARC forbids use of 'dealloc' in a @selector

哦！作者终于明白朋友口中的hook不住，其实就是因为写法的问题，Xcode不让我们这么写，所以谈何去hook呢?既然不能这么写那我们换种写法不就行了吗？是的。哈哈

```
Method oriMethod = class_getInstanceMethod([self class], NSSelectorFromString(@"dealloc"));

```
我们通过字符串获取到方法就可以了，这样Xcode在编译的时候检查不到啦。

```Objective-C
+(void)load{
//    Method oriMethod = class_getInstanceMethod([self class], @selector(dealloc));

    Method oriMethod = class_getInstanceMethod([self class], NSSelectorFromString(@"dealloc"));
    Method repMethod = class_getInstanceMethod([self class], @selector(replaceDelloc));
    method_exchangeImplementations(oriMethod, repMethod);
}
- (void)replaceDelloc{
    NSLog(@"delloc被替换了");
}
```
