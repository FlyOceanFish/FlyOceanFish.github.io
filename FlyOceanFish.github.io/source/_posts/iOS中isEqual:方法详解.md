---
title: iOS中isEqual:方法详解 # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
## 背景
　　大家对于这个方法其实再熟悉不过了，比较对象的时候用这个方法，如果使用==针对对象则仅仅比较的是内存地址的引用。但是大家可能会忽略另一个细节，对象的hash值

引用苹果官方的一句话：

    This method defines what it means for instances to be equal. For example, a container object might define two containers as equal if their corresponding objects all respond YES to an isEqual:  request. See the NSData, NSDictionary, NSArray, and NSString class   specifications for examples of the use of this method.

    If two objects are equal, they must have the same hash value. This last point is particularly important if you define isEqual: in a subclass and intend to put instances of that subclass into a collection. Make sure you also define hash in your subclass.
# #自定义对象，重写isEqual方法
``` Objective-C
@implementation Person
- (instancetype)initWithName:(NSString *)name andSex:(NSString *)sex{
    self = [super init];
    if (self) {
        self.name = name;
        self.sex = sex;
    }
    return self;
}
- (BOOL)isEqual:(id)object {
    if (self == object) {
        return YES;
    }

    if (![object isKindOfClass:[Person class]]) {
        return NO;
    }

    return [self isEqualToPerson:(Person *)object];
}
- (BOOL)isEqualToPerson:(Person *)person {
    if (!person) {
        return NO;
    }

    BOOL haveEqualNames = (!self.name && !person.name) || [self.name isEqualToString:person.name];
    BOOL haveEqualSex = (!self.sex && !person.sex) || [self.sex isEqualToString:person.sex];

    return haveEqualNames && haveEqualSex;
}
-(NSUInteger)hash{
    return self.name.hash^self.sex.hash;
}
@end
```

上边代码是我定义的Person类并且重写了isEqual方法，大家可能发现了我还重写了hash方法。如果不重写hash方法会出现什么问题呢？？

正常比较两个对象是否相等已经够用了，但是如果我们使用NSSet等等这样与hash相关的类就会产生问题了。
```Objective-C
    Person *person1 = [[Person alloc] initWithName:@"ocean" andSex:@"boy"];
    Person *person2 = [[Person alloc] initWithName:@"ocean" andSex:@"boy"];
    NSSet *set = [NSSet setWithObjects:person1,person2, nil];
    NSLog(@"set---%@",set);
    if ([person1 isEqual:person2]) {
        NSLog(@"相等");
    }else{
        NSLog(@"不相等");
    }
```
如果我们不重写hash方法set打印出来后会有两个对象，是由于系统默认给我们实现了一个hash方法，perons1和person2的hash值不相等。如果重写了则只会打印出一个对象。

系统默认实现
```Objective-C
-(NSUInteger)hash{
    return (NSUInteger)self
}
```

##总结
如果两个对象相等，则hash必须要相等
##参考
[Equality](http://nshipster.com/equality/)
[Apple Developer Documentation](https://developer.apple.com/documentation/objectivec/1418956-nsobject/1418795-isequal?language=objc)
[Implementing Equality and Hashing](https://www.mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html)
