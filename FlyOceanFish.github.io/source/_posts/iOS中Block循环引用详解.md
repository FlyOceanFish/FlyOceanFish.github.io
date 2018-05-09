---
title: iOS中Block循环引用刨根问底 # 这是标题
date: 2018-05-07 10:04
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 序言
Blocks是苹果出的轻量型回调方式，使用起来既简洁，又方便。不过就是会产生一个问题：循环引用。进而会导致内存释放不了，造成内存泄漏。那到底怎么样才会产生循环引用呢?如何解决呢？

这篇文章我们就用多个案例从本质上去解析到底啥是循环引用

# 案例解析
```Objective-C
typedef void(^Blk_t)(void);
```

## 案例1
我们首先先来看一个循环引用的案例
声明一个全局变量
```Objective-C
@property (nonatomic,copy) Blk_t blk;
```

```Objective-C
- (void)cycleRetainMethod1
{
    self.blk = ^void(void){
        [self doSomething];
    };
    self.blk();
}
```
在UIViewController中调用这个方法，则会导致循环引用。

接下来让我们分析为什么会循环引用，看下图：
![](https://user-gold-cdn.xitu.io/2018/5/7/16338723f5a97ab7?w=562&h=229&f=jpeg&s=28454)
分析循环引用其实只要通过以上图都可以分析出来是否产生了循环引用。很典型就是相互持有产生了一个闭环，最终导致谁也释放不了，从而内存泄漏。
一般我们是通过以下方法解决：

```Objective-C
- (void)cycleRetainMethod1
{
    __weak typeof(self) this = self;
    self.blk = ^void(void){
        [this doSomething];
    };
    self.blk();
}
```
通过持有即可解决循环引用。


## 案例2
```Objective-C
- (void)cycleRetainMethod2
{

    Blk_t block = ^ void(void){
        [self doSomething];
    };
    block();
}
```
当我们在UIViewController调用这个方法的时候是否会产生循环引用呢？

答案是**不会的**

我们也是通过以上的图示来画一个
![](https://user-gold-cdn.xitu.io/2018/5/7/163387962a38cf20?w=559&h=208&f=jpeg&s=27557)
因为block变量是局部变量，所以UIViewController是不会持有的，从而导致原来的闭环断开，所以不会造成内存泄漏。
## 案例3
这个案例是一个对比案例，如果通过简单的画图可能不足余说明问题，所以我们结合编译后的源码就可以彻底说明

Person.m文件
```Objective-C
@implementation Person

- (instancetype)initWithBlock:(Block)block
{
    if (self = [super init]) {
        _blk = block;
    }
    return self;
}

- (void)Block:(Block)block
{
    _blk = block;
    NSLog(@"====%@",block);
}


- (void)execute
{
    _blk(@"回调数据");
}

-(void)dealloc{
    NSLog(@"person释放了");
}
@end

```
* 第一种写法

```Objective-C
- (void)cycleRetainMethod3
{
    //第一种写法
    _person = [[Person alloc] initWithBlock:^(NSString *str) {
        [self doSomething];

    }];
    [person execute];

}
```
* 第二种写法

```Objective-C
- (void)cycleRetainMethod3
{
  //第二种写法
  _person2 = [[Person alloc] init];
  [person2 Block:^(NSString *str) {
      [self doSomething];
  }];
  [person2 execute];
}

```
以上是代码，大家觉着会产生循环引用吗？
答案是：第一种写法不会产生循环引用，第二种会产生循环引用

第二种通过案例1的图示很容易就能看出来这里产生了循环引用。
那第一种为啥没产生循环引用呢？

这里我们通过对比分析这两个写法的源码来看

第一种写法Block实现的代码
```C
struct __SecondViewController__cycleRetainMethod55_block_impl_0 {
  struct __block_impl impl;
  struct __SecondViewController__cycleRetainMethod55_block_desc_0* Desc;
  SecondViewController *self;
  __SecondViewController__cycleRetainMethod55_block_impl_0(void *fp, struct __SecondViewController__cycleRetainMethod55_block_desc_0 *desc, SecondViewController *_self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
第二种写法Block实现的代码
```C
struct __SecondViewController__cycleRetainMethod5_block_impl_0 {
  struct __block_impl impl;
  struct __SecondViewController__cycleRetainMethod5_block_desc_0* Desc;
  SecondViewController *self;
  __SecondViewController__cycleRetainMethod5_block_impl_0(void *fp, struct __SecondViewController__cycleRetainMethod5_block_desc_0 *desc, SecondViewController *_self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
大家对比两个源码有没发现？第二种写法Block的实现引用了`SecondViewController *self`,而第一种写法是没有引用的！
>也就是说第一种写法，UIViewController是持有Block的，而Block是不持有UIViewController的，所以不会造成循环引用。

其实为啥不会引用UIViewController呢？大家看Person.m的`- (instancetype)initWithBlock:(Block)block`的方法实现，在返回`self`之前对变量`_blk`赋值了。即`_blk`变量已经赋值，但是此时`_person`变量还没有产生。
# 总结
Block循环引用其实分析起来很简单，不管中间经过多少变量，只要一层一层的分析，看看有没有持有，画一个简单的引用图即可，是闭环那就是循环引用。而且我们通过编译之后的源码也能非常清楚的知道了什么叫持有了吧。
