---
title: 你真的了解iOS怎么取属性的吗？ # 这是标题
date: 2017-12-28 13:10:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---

如果iOS中谈到取属性，相信大家都会夸夸其谈，不就是get方法吗？或者大谈kvc取属性的机制。不得不说这些也是对的。这时大家可能就疑惑了，那你还要说啥的！！大家不妨想想，这些都是代码层的实现，其实我们的代码最终都会被编译，然后加载到内存中，那你在内存中是怎么取到属性的呢？？对的我们讨论就是它！
## 指针
如果说到内存，不知道大家会不会想到**指针**呢？这里简单介绍一下，让大家有个简单的理解。如果理解不了的话，建议大家找一个C语言的教程，学一下指针。

>指针（Pointer）是编程语言中的一个对象，利用地址，它的值直接指向（points to）存在电脑存储器中另一个地方的值。由于通过地址能找到所需的变量单元，可以说，地址指向该变量单元。因此，将地址形象化的称为“指针”。意思是通过它能找到以它为地址的内存单元。

* 那到底什么是指针呢？？
> 类型 * 变量名

这就是声明了一个指针变量

* 指针类型有什么作用呢？

比如：
````
int* num;
````
<font color='red'>指针变量的类型决定了通过这个指针找到变量的首地址以后，连续操作多少个字节空间</font>
为什么会说连续操作多少个字节空间？？主要是指针有算术运算加减，说白了就是指针的移动。

* 指针是int* 连续操作4个字节

* 指针是double* 连续操作8个字节

比如:
```
int* p = &num;
p++;
```
当指针+1的时候，这时候指针要移动1个单元，而不是1个字节！！
那到底这1个单元是多大呢？其实1个单元的大小就是指针类型的大小。这里是`int`型，所以移动了4个字节

---
以上就是简单给大家做了**指针**介绍，其实理解了指针，对于我们出现的一些野指针的bug、runtime源码中的一些机制等等是有所帮助的。言归正传。接下来让我看一道题，真正的去了解内存和指针的关系。

```
int num1 = 10;
int num = 20;
int* p = &num;
p++;
printf("%d\n",*p);//打印为10，因为p++，指针已经移动了4个字节，下一个内存存储10正好是4个字节
```
这里其实是前边声明了一个num1，正好是4个字节，所以就将10取出来了。（说白了就是内存中下一个连续的4个字节存的是什么取出来就是什么）

说了这么多都是指针和内存，建议大家搞明白以上内容再读以下的内容，如果上边都搞不明白的话，下边有关iOS中runtime取属性的内容有可能就会云里雾里。

## iOS中成员变量与属性

以下题目是sunnyxx习题中的一题，网上也有详细的[答案](http://blog.csdn.net/shznt/article/details/50481819)。这里作者就简述一下自己的理解，如果想看非常详细的答案的话可以点击上边的链接。

下面代码会? Compile Error / Runtime Crash / NSLog…?

```
@interface Sark : NSObject

@property (nonatomic, copy) NSString *name;

@end

@implementation Sark

- (void)speak

{

    NSLog(@"my name is %@", self.name);

}

@end

@interface Test : NSObject

@end

@implementation Test

- (instancetype)init

{

    self = [super init];

    if (self) {

        id cls = [Sark class];

        void *obj = &cls;

        [(__bridge id)obj speak];

    }

    return self;

}

@end

int main(int argc, const char * argv[]) {

    @autoreleasepool {

        [[Test alloc] init];

    }

    return 0;

}
```
答案：代码正常输出，输出结果为：

>2014-11-07 14:08:25.698 Test[1097:57255] my name is

* 为什么能够正常运行，并调用到speak方法？

计算机将我们的`Sark`类信息通过
`id cls = [Sark class];`这一行加载到内存中，并且取得了`cls`变量。这个时候其实我们只要知道`cls`这个变量的地址就行了,其实相当于类的对象的地址。`void *obj = &cls;`这句话就让我们获得了对象的地址。(平时我们`new`对象的时候就干了两件事：1、申请内存；2、获取内存的地址(对象变量的地址就是内存的地址)，这里的对象与我们`new`出来的对象有所不同。但是虽然不是new对象,iOS中`Class`对象已经存储了我们需要的东西。比如有关变量的内存**偏移**、方法等等所有的信息）接下来可以干我们想干的任何事情了。
>iOS中`Class`中存储了我们想要的东西，这一块的知识要上升到了runtime的源码，上边给到的链接中有详细介绍。其实大家想想编译完之后肯定得有一个类或者其他东西存储着有关内存等等相关的信息的。

* 为什么self.name会输出?

我们程序在编译之后其实就是一堆的汇编指令，汇编操作的就是**内存地址**。所以当我们程序运行的时候都是**寄存器**一条条的执行汇编指令。其实执行汇编指令最重要的就是变量、方法、对象等等的一大堆地址，因为寄存器有限，所以会把有限的数据从内存中加载到寄存器。所以总得来说是操作寄存器的地址和内存地址。如果没有地址那怎么知道执行什么呢？所以只要有地址了就好办了。

指令如下图：
![image.png](http://upload-images.jianshu.io/upload_images/6644906-477a0927a87fd05e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

变量对应于runtime的objc_ivar代码如下:
```
struct objc_ivar {

    char *ivar_name                                          OBJC2_UNAVAILABLE;

    char *ivar_type                                          OBJC2_UNAVAILABLE;

    int ivar_offset                                          OBJC2_UNAVAILABLE;

#ifdef __LP64__

    int space                                                OBJC2_UNAVAILABLE;

#endif

}   
```
其中 `ivar_offset`就是变量的地址偏移字节。

>变量地址=对象地址 ＋ 基类大小 + ivar偏移字节

到这里再结合我上边指针的铺垫相信大家应该明白了为什么为什self.name会输出吧。

其实通过这里我们也知道了其实iOS中取对象就是指针的偏移。

```
Student *student = [[Student alloc] init];

  Ivar age_ivar = class_getInstanceVariable(object_getClass(student), "age");

  int *age_pointer = (int *)((__bridge void *)(student) + ivar_getOffset(age_ivar));

  NSLog(@"age ivar offset = %td", ivar_getOffset(age_ivar));

  *age_pointer = 10;

  NSLog(@"%@", student);
```
