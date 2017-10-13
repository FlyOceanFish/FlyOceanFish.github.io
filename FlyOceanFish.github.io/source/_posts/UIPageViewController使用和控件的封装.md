---
title: UIPageViewController使用和控件的封装 # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
## 背景
我们经常会遇到一个场景，就是同时有几个视图需要左右滑动显示。大家可能会通过UIScrollview来实现，也能实现。但是如果不自己实现一些懒加载、缓存其实对性能还是有浪费的。接下来我们就让U闪亮登场。UIPageViewController加载不同的UIViewController可以有效的将不同逻辑进行了分离，减轻了总UIViewController的负担。
我通过UIPageViewController简单封装了一个工具，就是加载不同的视图，支持上下滑动、左右滑动。并且具有懒加载和缓存的功能，在一定程度上提高了性能。
#### 初始化
``` Objective-C
    self.array = @[[FirstViewController class],[SecondViewController class],[FirstViewController class]];
//实例化UIPageViewController，并且指定方向和动画。可以设置options。UIPageViewControllerOptionInterPageSpacingKey两个视图的间距
    UIPageViewController *page = [[UIPageViewController alloc] initWithTransitionStyle:UIPageViewControllerTransitionStyleScroll navigationOrientation:UIPageViewControllerNavigationOrientationHorizontal options:@{UIPageViewControllerOptionInterPageSpacingKey:@10}];
    page.delegate = self;
    page.dataSource = self;
````
````
typedef NS_ENUM(NSInteger, UIPageViewControllerTransitionStyle) {
    UIPageViewControllerTransitionStylePageCurl = 0, //翻书的效果
    UIPageViewControllerTransitionStyleScroll = 1 // 正常滑动切换效果
};
 ````
````
typedef NS_ENUM(NSInteger, UIPageViewControllerNavigationDirection) {
    UIPageViewControllerNavigationDirectionForward,
    UIPageViewControllerNavigationDirectionReverse
};
其实这两个就是一对。如果是左右滑动的话，一个代表从左到右，另一个就是从右到左；如果上下滑动，一个代表从上到下，另一个就是从下到上。
````
````
//设置当前显示ViewController,这里通过一个方法集中生成，其实也是做到缓存功能。
    [page setViewControllers:@[[self private_viewControllerForPage:0]] direction:UIPageViewControllerNavigationDirectionForward animated:YES completion:nil];
    [self addChildViewController:page];
    [self.view addSubview:page.view];
    [page didMoveToParentViewController:self];
````
````
//实例化UIViewController,并且缓存起来
- (UIViewController *)private_viewControllerForPage:(NSUInteger)pageIndex{
    if (pageIndex<self.pageViewControllerClasses.count) {
        if (pageIndex<self.cacheViewControllerArray.count) {
            return self.cacheViewControllerArray[pageIndex];
        }else{
            Class controllerClass =  self.pageViewControllerClasses[pageIndex];
            UIViewController *controller = [[controllerClass alloc] init];
            [self.cacheViewControllerArray insertObject:controller atIndex:pageIndex];
            return controller;
        }
    }
    return nil;

}
- (NSInteger)private_indexOfViewController:(UIViewController *)viewController{
    NSInteger index = [self.cacheViewControllerArray indexOfObject:viewController];
    return index;
}
````
#### 代理方法
````
//向后滑动的时候，调用此方法。获取之前显示过的UIViewController
- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerBeforeViewController:(UIViewController *)viewController{
    NSInteger currentIndex = [self private_indexOfViewController:viewController];
    if (currentIndex>0) {
        return [self private_viewControllerForPage:currentIndex-1];
    }
    return nil;
}
//向前滑动的时候，调用此方法。获取之后要显示的UIViewController
- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerAfterViewController:(UIViewController *)viewController{
    NSInteger currentIndex = [self private_indexOfViewController:viewController];
    if (currentIndex>=0) {
        return [self private_viewControllerForPage:currentIndex+1];
    }
    return nil;
}
//滑动结束后回调此方法
- (void)pageViewController:(UIPageViewController *)pageViewController didFinishAnimating:(BOOL)finished previousViewControllers:(NSArray<UIViewController *> *)previousViewControllers transitionCompleted:(BOOL)completed{
    if (completed&&self.YTOPageViewControllerPageChanged) {
        self.YTOPageViewControllerPageChanged([self private_indexOfViewController:pageViewController.viewControllers.firstObject]);
    }
}
````

上边就是UIPageViewController的基本用法。大家可以自行去探索一下。其实挺好玩的，也可以实现一下显示两个视图，看起来就是一本打开的书的效果。
下边是我封装的一个工具类，大家可以参考一下。只要传入你需要显示的UIViewController
的class名字就可以。
## 效果图
![效果.png](http://upload-images.jianshu.io/upload_images/6644906-c23a7e673aca129a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 控件地址
***大家觉着有用别忘了star一下哦！***
[YTOSectionsViewController](https://github.com/FlyOceanFish/YTOSectionsViewController)
