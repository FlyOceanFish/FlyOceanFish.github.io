---
title: iOS实现微信列表长按效果 # 这是标题
date: 2017-11-07 15:12:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---

不知道大家有没有发现微信聊天历史列表、苹果自带的通讯录历史记录等应用如隐藏的长按功能。哈哈，如果没有也不要气馁，也不要怀疑人生。要不就要换手机了，要不就要升级系统了！

这个功能看着挺炫的，也挺方便的吧，如果看历史记录等等其他快捷操作的一个入口。如果没有想到正确的实现方法，可能会实现起来很难，或者效果没有这么好。如果找到了正确的方法，其实实现起来非常简单。

其实这个效果就是利用了苹果的3Dtouch，不支持3Dtouch的手机或系统就没法体验到了。

卖了一个大关子，也索罗了一顿，接下来让我看看如何实现的。哈哈。

### 给需要的UIView注册3DTouch行为

```Objective-C
[self registerForPreviewingWithDelegate:self sourceView:cell];
```
这个方法其实就是`UIViewController`的方法，最后一个参数就是指定需要添加3DTouch的View，这里我们要加的是我们自己的tableviewcell.

```Objective-C
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    MYTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"MYTableViewCell" forIndexPath:indexPath];
    [self registerForPreviewingWithDelegate:self sourceView:cell];
    return cell;
}

```
### 实现`UIViewControllerPreviewingDelegate`协议

这个协议有两个方法我们都要实现
* 第一个`- (nullable UIViewController *)previewingContext:(id <UIViewControllerPreviewing>)previewingContext viewControllerForLocation:(CGPoint)location`

```Objective-C
- (nullable UIViewController *)previewingContext:(id <UIViewControllerPreviewing>)previewingContext viewControllerForLocation:(CGPoint)location{
    PreViewController *vc = [[PreViewController alloc] init];
    return vc;
}

```
这个协议方法就是我们3DTouch之后要显示的视图，想显示什么就自己实现，完全可以由自己来控制

* 第二个`- (void)previewingContext:(id <UIViewControllerPreviewing>)previewingContext commitViewController:(UIViewController *)viewControllerToCommit`

```Objective-C
- (void)previewingContext:(id <UIViewControllerPreviewing>)previewingContext commitViewController:(UIViewController *)viewControllerToCommit{
    [self showViewController:viewControllerToCommit sender:self];
}
```

这个协议方法就是将我们的视图显示出来

----
华丽的分割线，到这里基本就实现了长按弹出视图的效果，细心的人可能发现其实还有一个效果就是长按出现视图之后可以向上滑动出现，之后出现了几个按钮。接下来让我们看看怎么加上那几个按钮。

其实加这几个按钮也非常简单。找到我们之前定义的`PreViewController`这个类，然后重写`-(NSArray<id<UIPreviewActionItem>> *)previewActionItems` 这个方法也是`UIViewController`中本身自带的。

```Objective-C
-(NSArray<id<UIPreviewActionItem>> *)previewActionItems
{
    UIPreviewAction *p1 = [UIPreviewAction actionWithTitle:@"删除" style:UIPreviewActionStyleDefault handler:^(UIPreviewAction * _Nonnull action, UIViewController * _Nonnull previewViewController) {
        NSLog(@"点击了删除");
    }];


    UIPreviewAction *p2 = [UIPreviewAction actionWithTitle:@"撤销" style:UIPreviewActionStyleSelected handler:^(UIPreviewAction * _Nonnull action, UIViewController * _Nonnull previewViewController) {
        NSLog(@"点击了撤销");
    }];

    return @[p1,p2];
}
```
