---
title: iOS自己实现KVO # 这是标题
date: 2018-3-01 09:52:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---

# [KVO](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)是什么
一下是摘自苹果官网的一段话
>Key-value observing is a mechanism that allows objects to be notified of changes to specified properties of other objects

KVO其实就是可以让开发人员观察到对象特定属性变化的一种机制
> 重要一点:为了能理解KVO，首先要理解KVC。（<font color="red">因为KVC一定会触发KVO的，如果一个属性没有setXX方法，通过KVC改变属性，同样会触发KVO，所以KVO与属性的setXX方法没有必然联系</font>）

# 苹果内部实现细节
>Automatic key-value observing is implemented using a technique called isa-swizzling.

自动KVO是通过isa-swizzling技术来实现的，所以本文对KVO的实现是对自动KVO的实现，而且也正是苹果官网所说的isa-swizzling技术来实现
>When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.

当我们调用方法增加一个属性增加观察的时候，此时苹果系统会通过runtime生成一个原类的子类，名字为`NSKVONotifying_原类的名字`；同时对象的isa被指向了runtime自己生成的类。通过以下代码可以获得证明

```Objective-C
_p = [[Person alloc] init];
[_p addObserver:self forKeyPath:@"sex" options:NSKeyValueObservingOptionNew context:nil];
```
```Objective-C
-(void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
    NSLog(@"%@",change);
    Class class = object_getClass(_p);
    Class superClass = class_getSuperclass(class);
    NSLog(@"class---%@",_p.class);//这里打印出的还是Person，苹果做的伪装，此时我们可以通过调式控制台来观察
    NSLog(@"superClass:%@",superClass);
}
```
![1519874438299.jpg](http://upload-images.jianshu.io/upload_images/6644906-4217e48d51344443.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#  自己具体实现KVO
整个实现思路也主要就是上边的那些分析，通过runtime创建了一个子类，然后交换isa
```Objective-C
NSString *setterStr = private_setterForKey(key);

Method setterMethod = class_getInstanceMethod(self.class, NSSelectorFromString(setterStr));

NSString *oldClassName = NSStringFromClass(self.class);
NSString *kvoClassName = [@"FOFKVO_" stringByAppendingString:oldClassName];
Class kvoClass;
kvoClass = objc_lookUpClass(kvoClassName.UTF8String);
if (!kvoClass) {
    kvoClass = objc_allocateClassPair(self.class, kvoClassName.UTF8String, 0);
    objc_registerClassPair(kvoClass);
}

if (setterMethod) {//直接调用setXX方法改变值
    class_addMethod(kvoClass,NSSelectorFromString(setterStr), (IMP)setterIMP, "v@:@");//重写setXX方法
}else{//通过kvc改变值,通过method-swizzling
    Method method1 = class_getInstanceMethod(self.class, @selector(setValue:forKey:));
    Method method2 = class_getInstanceMethod(self.class, @selector(swizz_setValue:forKey:));
    method_exchangeImplementations(method1, method2);
}
object_setClass(self, kvoClass);//isa-swizzling实例的isa已经指向我们新创建的
FOFObserverInfo *info = [[FOFObserverInfo alloc] initWithObserver:observer.description key:key block:block];
NSMutableArray *observers = objc_getAssociatedObject(self, kObservers);
if (!observers) {
    observers = [NSMutableArray array];
    objc_setAssociatedObject(self, kObservers, observers, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
[observers addObject:info];
```

```Objective-C
#pragma mark - Private
#pragma mark swizz
-(void)swizz_setValue:(id)value forKey:(NSString *)key{
    id oldValue = [self valueForKey:key];
    [self swizz_setValue:value forKey:key];//如果这里没报错，说明正常设置值，现在开始回调
    NSMutableArray *observers = objc_getAssociatedObject(self, kObservers);
    for (FOFObserverInfo *temp in observers) {
        if ([temp.key isEqualToString:key]) {
            temp.block(self, key, oldValue, value);
        }
    }
}

#pragma mark overrid
void setterIMP(id self,SEL _cmd,id newValue){
    NSString *setterName = NSStringFromSelector(_cmd);
    NSString *temp = private_upperTolower([setterName substringFromIndex:@"set".length], 0);//去除set并将大写改成小写
    NSString *key = [temp substringToIndex:temp.length-1];//去除冒号
    id oldValue = [self valueForKey:key];
    struct objc_super superClazz = {
        .receiver = self,
        .super_class = class_getSuperclass(object_getClass(self))
    };
    ((void (*)(void *, SEL, id))objc_msgSendSuper)(&superClazz, _cmd, newValue);
    NSMutableArray *observers = objc_getAssociatedObject(self, kObservers);
    for (FOFObserverInfo *temp in observers) {
        if ([temp.key isEqualToString:key]) {
            temp.block(self, key, oldValue, newValue);
        }
    }
}
#pragma mark inline
static force_inline NSString * private_setterForKey(NSString *key){
    key = private_lowerToUpper(key, 0);
    return [NSString stringWithFormat:@"set%@:",key];
}

static force_inline NSString * private_lowerToUpper(NSString *str,NSInteger location){
    NSRange range = NSMakeRange(location, 1);
    NSString *lowerLetter = [str substringWithRange:range];
    return [str stringByReplacingCharactersInRange:range withString:lowerLetter.uppercaseString];
}
static force_inline NSString * private_upperTolower(NSString *str,NSInteger location){
    NSRange range = NSMakeRange(location, 1);
    NSString *lowerLetter = [str substringWithRange:range];
    return [str stringByReplacingCharactersInRange:range withString:lowerLetter.lowercaseString];
}
```
# 完整代码
[github完整代码地址](https://github.com/FlyOceanFish/NSObject-FOFKVO)
# 资料
以下两篇苹果官方文档很详细，建议大家要想仔细研究KVO和KVC的话，最好先仔细看一下以下两篇文档，这两篇文章基本上够用了。因为网上一些非官方的文档都是自己的理解，不一定对。

* [KVO](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)
* [KVC](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/index.html#//apple_ref/doc/uid/10000107-SW1)
