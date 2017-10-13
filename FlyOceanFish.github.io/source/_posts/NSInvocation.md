---
title: NSInvocation # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- Objective-C
---
NSInvocation是命令模式的一种实现，它包含选择器、方法签名、相应的参数以及目标对象。所谓的方法签名，即方法所对应的返回值类型和参数类型。当NSInvocatio被调用，它会在运行时通过目标对象去寻找对应的方法，从而确保唯一性，可以用[receiver message]来解释。实际开发过程中直接创建NSInvocation的情况不多见，这些事情通常交给系统来做。比如bang的JSPatch中arm64方法替换的实现就是利用runtime消息转发最后一步中的NSInvocation实现的。
###正文
基于这种命令模式，可以利用NSInvocation调用任意SEL甚至block。NSInvocation与NSMethodSignature配合来完成调用。NSMethodSignature这个类的对象保存了方法的名称、参数和返回值
- SEL
系统`NSObject`自带的`performSelector`默认只支持一个参数，我们可以给NSObject增加一个category，增加以下方法就可以支持多个参数的调用了

````
- (id)performSelector:(SEL)aSelector withArguments:(NSArray *)arguments {
    if (aSelector == nil) return nil;
    NSMethodSignature *signature = [[self class] instanceMethodSignatureForSelector:aSelector];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.target = self;
    invocation.selector = aSelector; // invocation 有2个隐藏参数，所以 argument 从2开始
    if ([arguments isKindOfClass:[NSArray class]]) {
        NSInteger count = MIN(arguments.count, signature.numberOfArguments - 2);
        for (int i = 0; i < count; i++) {
            const char *type = [signature getArgumentTypeAtIndex:2 + i]; // 需要做参数类型判断然后解析成对应类型，这里默认所有参数均为OC对象
            if (strcmp(type, "@") == 0) {
                id argument = arguments[i]; [invocation setArgument:&argument atIndex:2 + i];
            }
        }
    }
    [invocation invoke];
    id returnVal;
    if (strcmp(signature.methodReturnType, "@") == 0) {
        [invocation getReturnValue:&returnVal];
    } // 需要做返回类型判断。比如返回值为常量需要包装成对象，这里仅以最简单的`@`为例
    return returnVal;
}
````
![ED3CFE7D-C30A-4ADD-929D-B5F7CB806E82.png](http://upload-images.jianshu.io/upload_images/6644906-b09479039ddf5488.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![FCB4F1F5-1F15-4FDC-A3AE-C1DC95AF32BF.png](http://upload-images.jianshu.io/upload_images/6644906-c1af7b09f14982c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###block
````
static id invokeBlock(id block ,NSArray *arguments) {
    if (block == nil) return nil;
    id target = [block  copy];

    const char *_Block_signature(void *);
    const char *signature = _Block_signature((__bridge void *)target);

    NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:signature];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    invocation.target = target;

    // invocation 有1个隐藏参数，所以 argument 从1开始
    if ([arguments isKindOfClass:[NSArray class]]) {
        NSInteger count = MIN(arguments.count, methodSignature.numberOfArguments - 1);
        for (int i = 0; i < count; i++) {
            const char *type = [methodSignature getArgumentTypeAtIndex:1 + i];
            NSString *typeStr = [NSString stringWithUTF8String:type];
            if ([typeStr containsString:@"\""]) {
                type = [typeStr substringToIndex:1].UTF8String;
            }

            // 需要做参数类型判断然后解析成对应类型，这里默认所有参数均为OC对象
            if (strcmp(type, "@") == 0) {
                id argument = arguments[i];
                [invocation setArgument:&argument atIndex:1 + i];
            }
        }
    }

    [invocation invoke];

    id returnVal;
    const char *type = methodSignature.methodReturnType;
    NSString *returnType = [NSString stringWithUTF8String:type];
    if ([returnType containsString:@"\""]) {
        type = [returnType substringToIndex:1].UTF8String;
    }
    if (strcmp(type, "@") == 0) {
        [invocation getReturnValue:&returnVal];
    }
    // 需要做返回类型判断。比如返回值为常量需要包装成对象，这里仅以最简单的`@`为例
    return returnVal;
}
````



![9E8A9D90-F9A9-4DD3-AA21-9953E4953C8D.png](http://upload-images.jianshu.io/upload_images/6644906-f5230ba3653b8382.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/6644906-bf7381308a1d6b66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



###SEL与block比较
 > -  invocation
SEL既有target也有selector，block只有target
>- signature
>SEL有两个隐藏参数，类型均为@，分别对应target和selector。block有一个隐藏参数，类型为@?，对应target且block的target为他本身
>- type
以OC对象为例：SEL的type为@，block的type会跟上具体类型，如@"NSString"

###再谈block
在block的invocation中有这样的代码
````
const char *_Block_signature(void *);
const char *signature = _Block_signature((__bridge void *)target);
````
_Block_signature其实是JavaScriptCore/ObjcRuntimeExtras.h
中的私有API(这个头文件并没有公开[可以戳这里查看](http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/API/ObjcRuntimeExtras.h))

既然苹果把API封了，那就自己实现咯，万能的github早有答案[CTObjectiveCRuntimeAdditions](https://github.com/ebf/CTObjectiveCRuntimeAdditions)把CTBlockDescription.h
和CTBlockDescription.m
拖到项目中，代码这样写
````
static id invokeBlock(id block ,NSArray *arguments) {
    if (block == nil) return nil;
    id target = [block  copy];

    CTBlockDescription *ct = [[CTBlockDescription alloc] initWithBlock:target];
    NSMethodSignature *methodSignature = ct.blockSignature;
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    invocation.target = target;

    // invocation 有1个隐藏参数，所以 argument 从1开始
    if ([arguments isKindOfClass:[NSArray class]]) {
        NSInteger count = MIN(arguments.count, methodSignature.numberOfArguments - 1);
        for (int i = 0; i < count; i++) {
            const char *type = [methodSignature getArgumentTypeAtIndex:1 + i];
            NSString *typeStr = [NSString stringWithUTF8String:type];
            if ([typeStr containsString:@"\""]) {
                type = [typeStr substringToIndex:1].UTF8String;
            }

            // 需要做参数类型判断然后解析成对应类型，这里默认所有参数均为OC对象
            if (strcmp(type, "@") == 0) {
                id argument = arguments[i];
                [invocation setArgument:&argument atIndex:1 + i];
            }
        }
    }

    [invocation invoke];

    id returnVal;
    const char *type = methodSignature.methodReturnType;
    NSString *returnType = [NSString stringWithUTF8String:type];
    if ([returnType containsString:@"\""]) {
        type = [returnType substringToIndex:1].UTF8String;
    }
    if (strcmp(type, "@") == 0) {
        [invocation getReturnValue:&returnVal];
    }
    // 需要做返回类型判断。比如返回值为常量需要包装成对象，这里仅以最简单的`@`为例
    return returnVal;
}
````
![17D15BD3-98AA-46E9-BAD4-2747E16DD677.png](http://upload-images.jianshu.io/upload_images/6644906-143e6ac30788e9c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###NSInvocation.h
````
@interface NSInvocation : NSObject {
@private
    __strong void *_frame;
    __strong void *_retdata;
    id _signature;
    id _container;
    uint8_t _retainedArgs;
    uint8_t _reserved[15];
}

// 通过NSMethodSignature对象创建NSInvocation对象，NSMethodSignature为方法签名类
+ (NSInvocation *)invocationWithMethodSignature:(NSMethodSignature *)sig;

// 获取NSMethodSignature对象
@property (readonly, retain) NSMethodSignature *methodSignature;

// 保留参数，它会将传入的所有参数以及target都retain一遍
- (void)retainArguments;

// 判断参数是否还存在
// 调用retainArguments之前，值为NO，调用之后值为YES
@property (readonly) BOOL argumentsRetained;

// 设置消息调用者，注意：target最好不要是局部变量
@property (nullable, assign) id target;

// 设置要调用的消息
@property SEL selector;

// 获取消息返回值
- (void)getReturnValue:(void *)retLoc;

// 设置消息返回值
- (void)setReturnValue:(void *)retLoc;

// 获取消息参数
- (void)getArgument:(void *)argumentLocation atIndex:(NSInteger)idx;

// 设置消息参数
- (void)setArgument:(void *)argumentLocation atIndex:(NSInteger)idx;

// 发送消息，即执行方法
- (void)invoke;

// target发送消息，既设置了target同时执行方法。这一步要注意在调用此方法之前必须设置了selector和argument
- (void)invokeWithTarget:(id)target;

@end
````
