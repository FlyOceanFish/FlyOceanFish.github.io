
---
title: iOS读YYModel源码 # 这是标题
date: 2018-6-5 17:18:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 序文
在众多的json转model的第三方库中，YYModel可谓是表现出了优异的性能，那到底它是底层是如何实现的呢？带着这份好奇心，开启了我们读取YYModel源码之路
# 源码解读
## YYModel类图
此图可以点击放大，查看高清图。

![](https://upload-images.jianshu.io/upload_images/6644906-17dcf1bbbd2593f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过以上类图，我们可以清晰的看到了YYModel整个框架的设计，不得不说设计的非常的棒。

* `NSObject`、`NSArray`、`NSDictionary`的category分别提供最上层api，以供我们json<->model；
* `YYModel`协议供我们在转换过程中的个性化，包括自定义映射、集合泛型、白名单、黑名单、自定义转换过程等等；

* `_YYModelMeta`和`_YYModelPropertyMeta` 这两个类算是对`YYClassInfo`上层业务应用的封装，专门处理转换中的个性化(`YYModel`协议)
* `YYClassInfo`存储了我们定义的model的信息。包括父类、父类信息、所有的方法、属性、变量。其中分别用`YYClassMethodInfo`、`YYClassPropertyInfo`、`YYClassIvarInfo`;（淋漓尽致的表现出了面向对象编程）

## Json转Model

```Objective-C
NSString *json = @"{\"uid\":123456,\"name\":\"Harry\",\"created\":\"1965-07-31T00:00:00+0000\",\"books\":[{\"name\":\"Harry Potte\",\"pages\":256}]}";
User *user = [User yy_modelWithJSON:json];
```
接下来我们通过以上小小的例子正式进入源码解读。

```Objective-C
+ (instancetype)yy_modelWithJSON:(id)json {
    NSDictionary *dic = [self _yy_dictionaryWithJSON:json];
    return [self yy_modelWithDictionary:dic];
}
```
将我们的json字符串转换成`NSDictionary`,调用的是系统`NSJSONSerialization`完成的

```Objective-C
+ (instancetype)yy_modelWithDictionary:(NSDictionary *)dictionary {
    if (!dictionary || dictionary == (id)kCFNull) return nil;
    if (![dictionary isKindOfClass:[NSDictionary class]]) return nil;

    Class cls = [self class];
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:cls];
    if (modelMeta->_hasCustomClassFromDictionary) {
        cls = [cls modelCustomClassForDictionary:dictionary] ?: cls;
    }

    NSObject *one = [cls new];
    if ([one yy_modelSetWithDictionary:dictionary]) return one;
    return nil;
}
```
这个方法主要实现了两个功能：
1.对我们定义的Model进行解析生成_YYModelMeta，其中里边使用了缓存，使得只会解析一遍；
2.生成一个实例model，根据`_YYModelMeta`存储的model信息和上边生成的Dictionary对实例model复制，并且返回
### 赋值过程
```Objective-C
- (BOOL)yy_modelSetWithDictionary:(NSDictionary *)dic {
    if (!dic || dic == (id)kCFNull) return NO;
    if (![dic isKindOfClass:[NSDictionary class]]) return NO;


    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:object_getClass(self)];
    if (modelMeta->_keyMappedCount == 0) return NO;

    if (modelMeta->_hasCustomWillTransformFromDictionary) {
        dic = [((id<YYModel>)self) modelCustomWillTransformFromDictionary:dic];
        if (![dic isKindOfClass:[NSDictionary class]]) return NO;
    }

    ModelSetContext context = {0};
    context.modelMeta = (__bridge void *)(modelMeta);
    context.model = (__bridge void *)(self);
    context.dictionary = (__bridge void *)(dic);


    if (modelMeta->_keyMappedCount >= CFDictionaryGetCount((CFDictionaryRef)dic)) {//第一种赋值
        CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);
        // 这里是多级keyPath映射的时候。比如
        //+ (NSDictionary *)modelCustomPropertyMapper {
        //    return @{@"desc" : @"ext.desc"};
        //}

        if (modelMeta->_keyPathPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_keyPathPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_keyPathPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
        //这里是多个key对应一个属性。比如:
        // + (NSDictionary *)modelCustomPropertyMapper {
        //     return @{
        //              @"name" : @[@"name",@"Name",@"AName"]
        //              };
        // }
        if (modelMeta->_multiKeysPropertyMetas) {
            CFArrayApplyFunction((CFArrayRef)modelMeta->_multiKeysPropertyMetas,
                                 CFRangeMake(0, CFArrayGetCount((CFArrayRef)modelMeta->_multiKeysPropertyMetas)),
                                 ModelSetWithPropertyMetaArrayFunction,
                                 &context);
        }
    } else {//第二种赋值
        CFArrayApplyFunction((CFArrayRef)modelMeta->_allPropertyMetas,
                             CFRangeMake(0, modelMeta->_keyMappedCount),
                             ModelSetWithPropertyMetaArrayFunction,
                             &context);
    }

    if (modelMeta->_hasCustomTransformFromDictionary) {
        return [((id<YYModel>)self) modelCustomTransformFromDictionary:dic];
    }
    return YES;
}
```
这里是有两个赋值逻辑：
1.model需要赋值的属性是否不小于Dictionary里的数据。通过遍历数据去赋值；
2.model需要赋值的属性是否小于Dictionary里的数据，比如白名单、黑名单等个性化可以导致此情况发生。通过遍历model的属性去赋值
> 这里使用了两种赋值逻辑，只用第二种的话也是可以的，这样分开处理，性能提升会高一些吧

***这个赋值过程主要两个核心遍历方法:***
```Objective-C
static void ModelSetWithDictionaryFunction(const void *_key, const void *_value, void *_context) {
    ModelSetContext *context = _context;
    __unsafe_unretained _YYModelMeta *meta = (__bridge _YYModelMeta *)(context->modelMeta);
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = [meta->_mapper objectForKey:(__bridge id)(_key)];
    __unsafe_unretained id model = (__bridge id)(context->model);
    while (propertyMeta) {
        if (propertyMeta->_setter) {
            ModelSetValueForProperty(model, (__bridge __unsafe_unretained id)_value, propertyMeta);
        }
        propertyMeta = propertyMeta->_next;
    };
}
```
这个方法有个单向链表数据的遍历。其实就是一个key对应多个model属性的时候产生的。比如：
> `+ (NSDictionary *)modelCustomPropertyMapper {
    return @{
             @"name":@"name",
             @"Firstname":@"name"
             };
}`

```Objective-C
static void ModelSetWithPropertyMetaArrayFunction(const void *_propertyMeta, void *_context) {
    ModelSetContext *context = _context;
    __unsafe_unretained NSDictionary *dictionary = (__bridge NSDictionary *)(context->dictionary);
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = (__bridge _YYModelPropertyMeta *)(_propertyMeta);
    if (!propertyMeta->_setter) return;
    id value = nil;

    if (propertyMeta->_mappedToKeyArray) {
        value = YYValueForMultiKeys(dictionary, propertyMeta->_mappedToKeyArray);
    } else if (propertyMeta->_mappedToKeyPath) {
        value = YYValueForKeyPath(dictionary, propertyMeta->_mappedToKeyPath);
    } else {
        value = [dictionary objectForKey:propertyMeta->_mappedToKey];
    }

    if (value) {
        __unsafe_unretained id model = (__bridge id)(context->model);
        ModelSetValueForProperty(model, value, propertyMeta);
    }
}
```
这个方法主要分为3中情况:
1.多个key对应一个属性
2.有多急keypath进行映射
3.正常一个key对应一个属性
#### 核心赋值
核心赋值主要通过判断各种类型进行赋值，这里做了一些兼容性转换操作等等，最主要的是通过底层`objc_msgSend`消息发送的方式进行赋值，所以效率极高。由于代码太长，我这里只复制一部分给大家看看，感兴趣的话可以具体看源码
```
static void ModelSetValueForProperty(__unsafe_unretained id model,
                                     __unsafe_unretained id value,
                                     __unsafe_unretained _YYModelPropertyMeta *meta) {
  if (meta->_isCNumber) {
      NSNumber *num = YYNSNumberCreateFromID(value);
      ModelSetNumberToProperty(model, num, meta);
      if (num) [num class]; // hold the number
  } else if (meta->_nsType) {
      if (value == (id)kCFNull) {
          ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, (id)nil);
      } else {
          switch (meta->_nsType) {
              case YYEncodingTypeNSString:
              case YYEncodingTypeNSMutableString: {
                  if ([value isKindOfClass:[NSString class]]) {
                      if (meta->_nsType == YYEncodingTypeNSString) {
                          ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, value);
                      } else {
                          ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, ((NSString *)value).mutableCopy);
                      }
                  } else if ([value isKindOfClass:[NSNumber class]]) {
                      ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                     meta->_setter,
                                                                     (meta->_nsType == YYEncodingTypeNSString) ?
                                                                     ((NSNumber *)value).stringValue :
                                                                     ((NSNumber *)value).stringValue.mutableCopy);
                  } else if ([value isKindOfClass:[NSData class]]) {
                      NSMutableString *string = [[NSMutableString alloc] initWithData:value encoding:NSUTF8StringEncoding];
                      ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, string);
                  } else if ([value isKindOfClass:[NSURL class]]) {
                      ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                     meta->_setter,
                                                                     (meta->_nsType == YYEncodingTypeNSString) ?
                                                                     ((NSURL *)value).absoluteString :
                                                                     ((NSURL *)value).absoluteString.mutableCopy);
                  } else if ([value isKindOfClass:[NSAttributedString class]]) {
                      ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                     meta->_setter,
                                                                     (meta->_nsType == YYEncodingTypeNSString) ?
                                                                     ((NSAttributedString *)value).string :
                                                                     ((NSAttributedString *)value).string.mutableCopy);
                  }
              } break;
              。。。。
}
```
# 总结
到这里整个json转model已经结束，这里没有具体分析`YYModel`是怎么将我们的model模型解析存储到`YYClassInfo`中的，这个会单独写一篇详细介绍。

通过以上代码可以看到YY的作者对代码性能的要求极高，并且大部分是C函数的调用，不用通过消息转发直接，所以整体性能肯定会高。向大神致敬！
