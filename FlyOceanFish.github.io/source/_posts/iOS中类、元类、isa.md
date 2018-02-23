---
title: iOS中类、元类、isa详解 # 这是标题
date: 2018-01-09 17:48:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
类相信大家都知道是什么，如果看过runtime的源码或者看过相关的文章对isa肯定也不陌生，不过元类(meta class)大家可能就比较陌生了。不过大家也不要担心，我会细细道来，让大家明白它到底是个什么东西。

先看一段大家非常熟悉的代码：
````Objective-C
Person *person = [[Person alloc] init];
````
为什么`Person`类名就能调用到`alloc`方法吗？到底怎么找到了`alloc`的方法了呢？
>1.首先，在相应操作的对象中的缓存方法列表中找调用的方法，如果找到，转向相应实现并执行。

>2.如果没找到，在相应操作的对象中的方法列表中找调用的方法，如果找到，转向相应实现执行

>3.如果没找到，去父类指针所指向的对象中执行1，2.

>4.以此类推，如果一直到根类还没找到，转向拦截调用，走消息转发机制。

>5.如果没有重写拦截调用的方法，程序报错。

上边是我从网上一篇文章摘录的查找`alloc`的方法的大体过程。如果是实例方法(声明以`-`开头)这个描述的换个过程还是可以的，不过如果是类方法（声明以`+`开头比如`alloc`方法）还是有所欠缺的！
## 元类

`元类`也是类，是描述`Class `类对象的类。
````Objective-C
Class aclass = [Person class];
````
所以`Class meta = object_getClass(aclass)`得到的就是元类，与`Class metaClass = objc_getMetaClass(name)`得到的结果一致。
```
Person *person = [[Person alloc] init];
Class class = [Person class];
Class a = object_getClass(person);
Class b = object_getClass(class);//元类
Class c = objc_getMetaClass(object_getClassName(person));//b与c一样都是元类
BOOL aBOOL = class_isMetaClass(a);//NO
BOOL bBOOL = class_isMetaClass(b);//YES
```
>一切皆对象。每一个对象都对应一个类。 `Person` 类就是`person`变量对象的类,换句话说就是`person`对象的isa指向`Person`对应的结构体的类;`aclass`也是对象，描述它的类就是元类，换句话说`aclass`对象的isa指向的就是`元类`。
**元类保存了类方法的列表**。当一个类方法被调用时,元类会首先查找它本身是否有该类方法的实现,如果没有则该元类会向它的父类查找该方法,直到一直找到继承链的头。（回答文章上边查找方法所欠缺的地方）

![图](https://upload-images.jianshu.io/upload_images/1120732-a1d642700bd37501.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

> <font color='red'>这张图是非常精髓的，直接诠释了元类和isa。大家可以一边阅读本文，一边回忆此图，多看几遍。</font>

## clang之后的代码

```Objective-C
static void OBJC_CLASS_SETUP_$_Cat(void ) {
	OBJC_METACLASS_$_Cat.isa = &OBJC_METACLASS_$_NSObject;//meta class的isa指向NSObject的metal class
	OBJC_METACLASS_$_Cat.superclass = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Cat.cache = &_objc_empty_cache;
	OBJC_CLASS_$_Cat.isa = &OBJC_METACLASS_$_Cat;//Cat 的Class对象的isa指向他的meta class
	OBJC_CLASS_$_Cat.superclass = &OBJC_CLASS_$_NSObject;
	OBJC_CLASS_$_Cat.cache = &_objc_empty_cache;
}
```
通过以上的clang之后的代码也可以看出isa的指向与图片的是相对应的
---
上边都是概念性质偏多，不知道大家理解的如何。现在看一个实例来具体介绍上边的内容。
## 代码示例
```Objective-C
//  Created by FlyOceanFish on 2018/1/9.
//  Copyright © 2018年 FlyOceanFish. All rights reserved.
//

#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface Person: NSObject
@end

@implementation Person
+ (void)printStatic{

}
- (void)print{
    NSLog(@"This object is %p.", self);
    NSLog(@"Class is %@, and super is %@.", [self class], [self superclass]);
    const char *name = object_getClassName(self);
    Class metaClass = objc_getMetaClass(name);
    NSLog(@"MetaClass is %p",metaClass);
    Class currentClass = [self class];
    for (int i = 1; i < 5; i++)
    {
        NSLog(@"Following the isa pointer %d times gives %p", i, currentClass);
            unsigned int countMethod = 0;
        NSLog(@"---------------**%d start**-----------------------",i);
        Method * methods = class_copyMethodList(currentClass, &countMethod);
        [self printMethod:countMethod methods:methods ];
        NSLog(@"---------------**%d end**-----------------------",i);
        currentClass = object_getClass(currentClass);
    }

    NSLog(@"NSObject's class is %p", [NSObject class]);
    NSLog(@"NSObject's meta class is %p", object_getClass([NSObject class]));
}
- (void)printMethod:(int)count methods:(Method *) methods{
    for (int j = 0; j < count; j++) {

        Method method = methods[j];

        SEL methodSEL = method_getName(method);

        const char * selName = sel_getName(methodSEL);

        if (methodSEL) {
            NSLog(@"sel------%s", selName);
        }
    }
}
@end

@interface Animal: NSObject
@end

@implementation Animal
- (void)print{
    NSLog(@"This object is %p.", self);
    NSLog(@"Class is %@, and super is %@.", [self class], [self superclass]);
    const char *name = object_getClassName(self);
    Class metaClass = objc_getMetaClass(name);
    NSLog(@"MetaClass is %p",metaClass);
    Class currentClass = [self class];
    for (int i = 1; i < 5; i++)
    {
        NSLog(@"Following the isa pointer %d times gives %p", i, currentClass);
        currentClass = object_getClass(currentClass);
    }

    NSLog(@"NSObject's class is %p", [NSObject class]);
    NSLog(@"NSObject's meta class is %p", object_getClass([NSObject class]));
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *person = [[Person alloc] init];
        Class class = [Person class];
        [person print];
//        printf("--------------------------------\n");
//        Animal *animal = [[Animal alloc] init];
//        [animal print];
    }
    return 0;
}

```

这个示例有两部分功能：
1. 大家只看`Person`的演示功能即可。
2. 观察Person和Animal两个对象的打印(打印方法名的可以注释掉，将main方法中的代码注释打开)

### `Person`的演示功能(不打印方法名称)

```Objective-C
This object is 0x100408400.
Class is Person, and super is NSObject.
MetaClass is 0x100001328
Following the isa pointer 1 times gives 0x100001350
Following the isa pointer 2 times gives 0x100001328
Following the isa pointer 3 times gives 0x7fffb9a4f0f0
Following the isa pointer 4 times gives 0x7fffb9a4f0f0
NSObject's class is 0x7fffb9a4f140
NSObject's meta class is 0x7fffb9a4f0f0
```

我们来观察isa到达过的地址的值：

- 对象的地址是 0x100408400.
- 类的地址是 0x100001350.
- 元类的地址是 0x100001328.
- 根元类（NSObject的元类）的地址是 0x7fffb9a4f0f0.


对于本次打印我们可以做出以下结论（可以再去看一遍上边那张精髓的图）：
- 对于3、4次打印相同，就是因为NSObject元类的类是它本身.
- 我们在实例化对象的时候，其实是创建了许多对象，这就是我们说的类簇。也对应了我们在用runtime创建类的时候`objc_allocateClassPair(xx,xx)`中是`ClassPair`而不是`bjc_allocateClass`
- 通过地址的大小也可以看出对象实例化先后，地址越小的越先实例化
- 很好的诠释了上边那张精髓图isa的指向
- NSObject的两个地址都非常大（哈哈哈哈哈！为什么非常大啊？？接下往下看）
### `Person`的演示功能(打印方法名称)

```Objective-C
Class is Person, and super is NSObject.
MetaClass is 0x100002378
Following the isa pointer 1 times gives 0x1000023a0
---------------**1 start**-----------------------
 sel------printMethod:methods:
sel------print
---------------**1 end**-----------------------
Following the isa pointer 2 times gives 0x100002378
---------------**2 start**-----------------------
sel------printStatic
---------------**2 end**-----------------------
Following the isa pointer 3 times gives 0x7fffb9a4f0f0
 ---------------**3 start**-----------------------
```
我只把重要的复制出来了，`NSObject`的所有的方法名没有复制出来，在此处不是重要的。

此次打印结果的结论：
- 类方法(静态方法)是存储在元类中的
### 观察Person和Animal两个对象的打印
```Objective-C
This object is 0x100508e70.
Class is Person, and super is NSObject.
MetaClass is 0x100001338
Following the isa pointer 1 times gives 0x100001360
Following the isa pointer 2 times gives 0x100001338
Following the isa pointer 3 times gives 0x7fffb9a4f0f0
Following the isa pointer 4 times gives 0x7fffb9a4f0f0
NSObject's class is 0x7fffb9a4f140
NSObject's meta class is 0x7fffb9a4f0f0
--------------------------------
This object is 0x100675ed0.
Class is Animal, and super is NSObject.
MetaClass is 0x100001388
Following the isa pointer 1 times gives 0x1000013b0
Following the isa pointer 2 times gives 0x100001388
Following the isa pointer 3 times gives 0x7fffb9a4f0f0
Following the isa pointer 4 times gives 0x7fffb9a4f0f0
NSObject's class is 0x7fffb9a4f140
NSObject's meta class is 0x7fffb9a4f0f0
Program ended with exit code: 0
```
此次打印的结论：
- `Animal`相关打印的地址都比`Person`的大。再次诠释了栈是由大往小排列的。栈口在最小的地方
- `Animal`和`Person`的`NSObject`的两个地址一样。(知道为什么大了吗？其实就是保证这两个地址足够大，以致于永远在栈中。这样整个程序中其实就是存在一个，有点像单例的意思)
