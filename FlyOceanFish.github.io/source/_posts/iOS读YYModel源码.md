
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
## 对象转NSDictionary
因为这里转换的结果提供给`yy_modelToJSONData`和`yy_modelToJSONString`这两个方法使用。并且这两个方法核心使用的是`NSJSONSerialization`的api。
以下是引用苹果[官方文档](https://developer.apple.com/documentation/foundation/nsjsonserialization)
>  
A Foundation object that may be converted to JSON must have the following properties:
>
>* The top level object is an `NSArray` or `NSDictionary`.
>
>* All objects are instances of `NSString`, `NSNumber`, `NSArray`, `NSDictionary`, or `NSNull`.
>* All dictionary keys are instances of `NSString`.
>
>* Numbers are not NaN or infinity.

所以整个转换过程是基于这个前提进行的转换，比如像NSURL、NSAttributedString、NSDate等类型则要转换成NSString。

核心方法就是以下代码：
```Objective-C
static id ModelToJSONObjectRecursive(NSObject *model) {
    if (!model || model == (id)kCFNull) return model;
    if ([model isKindOfClass:[NSString class]]) return model;
    if ([model isKindOfClass:[NSNumber class]]) return model;
    if ([model isKindOfClass:[NSDictionary class]]) {
        if ([NSJSONSerialization isValidJSONObject:model]) return model;
        NSMutableDictionary *newDic = [NSMutableDictionary new];
        [((NSDictionary *)model) enumerateKeysAndObjectsUsingBlock:^(NSString *key, id obj, BOOL *stop) {
            NSString *stringKey = [key isKindOfClass:[NSString class]] ? key : key.description;
            if (!stringKey) return;
            id jsonObj = ModelToJSONObjectRecursive(obj);
            if (!jsonObj) jsonObj = (id)kCFNull;
            newDic[stringKey] = jsonObj;
        }];
        return newDic;
    }
    if ([model isKindOfClass:[NSSet class]]) {
        NSArray *array = ((NSSet *)model).allObjects;
        if ([NSJSONSerialization isValidJSONObject:array]) return array;
        NSMutableArray *newArray = [NSMutableArray new];
        for (id obj in array) {
            if ([obj isKindOfClass:[NSString class]] || [obj isKindOfClass:[NSNumber class]]) {
                [newArray addObject:obj];
            } else {
                id jsonObj = ModelToJSONObjectRecursive(obj);
                if (jsonObj && jsonObj != (id)kCFNull) [newArray addObject:jsonObj];
            }
        }
        return newArray;
    }
    if ([model isKindOfClass:[NSArray class]]) {
        if ([NSJSONSerialization isValidJSONObject:model]) return model;
        NSMutableArray *newArray = [NSMutableArray new];
        for (id obj in (NSArray *)model) {
            if ([obj isKindOfClass:[NSString class]] || [obj isKindOfClass:[NSNumber class]]) {
                [newArray addObject:obj];
            } else {
                id jsonObj = ModelToJSONObjectRecursive(obj);
                if (jsonObj && jsonObj != (id)kCFNull) [newArray addObject:jsonObj];
            }
        }
        return newArray;
    }
    if ([model isKindOfClass:[NSURL class]]) return ((NSURL *)model).absoluteString;
    if ([model isKindOfClass:[NSAttributedString class]]) return ((NSAttributedString *)model).string;
    if ([model isKindOfClass:[NSDate class]]) return [YYISODateFormatter() stringFromDate:(id)model];
    if ([model isKindOfClass:[NSData class]]) return nil;


    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:[model class]];
    if (!modelMeta || modelMeta->_keyMappedCount == 0) return nil;
    NSMutableDictionary *result = [[NSMutableDictionary alloc] initWithCapacity:64];
    __unsafe_unretained NSMutableDictionary *dic = result; // avoid retain and release in block
    [modelMeta->_mapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyMappedKey, _YYModelPropertyMeta *propertyMeta, BOOL *stop) {
        if (!propertyMeta->_getter) return;

        id value = nil;
        if (propertyMeta->_isCNumber) {
            value = ModelCreateNumberFromProperty(model, propertyMeta);
        } else if (propertyMeta->_nsType) {
            id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
            value = ModelToJSONObjectRecursive(v);
        } else {
            switch (propertyMeta->_type & YYEncodingTypeMask) {
                case YYEncodingTypeObject: {
                    id v = ((id (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = ModelToJSONObjectRecursive(v);
                    if (value == (id)kCFNull) value = nil;
                } break;
                case YYEncodingTypeClass: {
                    Class v = ((Class (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = v ? NSStringFromClass(v) : nil;
                } break;
                case YYEncodingTypeSEL: {
                    SEL v = ((SEL (*)(id, SEL))(void *) objc_msgSend)((id)model, propertyMeta->_getter);
                    value = v ? NSStringFromSelector(v) : nil;
                } break;
                default: break;
            }
        }
        if (!value) return;

        if (propertyMeta->_mappedToKeyPath) {
            NSMutableDictionary *superDic = dic;
            NSMutableDictionary *subDic = nil;
            for (NSUInteger i = 0, max = propertyMeta->_mappedToKeyPath.count; i < max; i++) {
                NSString *key = propertyMeta->_mappedToKeyPath[i];
                if (i + 1 == max) { // end
                    if (!superDic[key]) superDic[key] = value;
                    break;
                }

                subDic = superDic[key];
                if (subDic) {
                    if ([subDic isKindOfClass:[NSDictionary class]]) {
                        subDic = subDic.mutableCopy;
                        superDic[key] = subDic;
                    } else {
                        break;
                    }
                } else {
                    subDic = [NSMutableDictionary new];
                    superDic[key] = subDic;
                }
                superDic = subDic;
                subDic = nil;
            }
        } else {
            if (!dic[propertyMeta->_mappedToKey]) {
                dic[propertyMeta->_mappedToKey] = value;
            }
        }
    }];

    if (modelMeta->_hasCustomTransformToDictionary) {
        BOOL suc = [((id<YYModel>)model) modelCustomTransformToDictionary:dic];
        if (!suc) return nil;
    }
    return result;
}
```
这个看成2步，第一步是返回NSArray/NSString/NSNumber/NSNull和基于苹果的要求对类型进行转换；第二步则将_YYModelMeta转换成NSDictionary，并且使用到了第一步的逻辑进行了递归。

在解析`_YYModelMeta`属性的时候也可分为两步：第一步判断_isCNumber、Foundation type和其他（YYEncodingTypeObject、ModelToJSONObjectRecursive、YYEncodingTypeSEL）；第二步则是对keypath映射的处理，即_mappedToKeyPath如果存在的话，则要相应生成subDic

对于Model转Json，Model转Data则是基于以上转换的过程。代码如下：
```Objective-C
- (NSData *)yy_modelToJSONData {
    id jsonObject = [self yy_modelToJSONObject];
    if (!jsonObject) return nil;
    return [NSJSONSerialization dataWithJSONObject:jsonObject options:0 error:NULL];
}

- (NSString *)yy_modelToJSONString {
    NSData *jsonData = [self yy_modelToJSONData];
    if (jsonData.length == 0) return nil;
    return [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
}

```
**这里提一个细节，如下代码：**
>for (NSUInteger i = 0, max = propertyMeta->_mappedToKeyPath.count; i < max; i++) {

再次感叹一下YY作者对于性能的极致追求！

# 总结
到这里整个json转model已经结束，这里没有具体分析`YYModel`是怎么将我们的model模型解析存储到`YYClassInfo`中的，这个会单独写一篇详细介绍。

通过以上代码可以看到YY的作者对代码性能的要求极高，并且大部分是C函数的调用，不用通过消息转发直接，所以整体性能肯定会高。向大神致敬！

通常Json转Model可以归纳为两种方法：一种是YY作者的基于消息发送，这种性能是比较高的；另一种则是基于KVC实现的，性能能差一些。
