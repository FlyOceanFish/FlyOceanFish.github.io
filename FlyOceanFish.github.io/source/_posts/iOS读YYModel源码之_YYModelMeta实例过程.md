---
title: iOS读YYModel源码之_YYModelMeta实例过程 # 这是标题
date: 2018-6-6 16:02:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 序文
上一篇[iOS读YYModel源码](http://flyoceanfish.top/2018/06/05/iOS%E8%AF%BBYYModel%E6%BA%90%E7%A0%81/)我们窥探了Json转Model的全过程，不过还差一步`_YYModelMeta`实例化的过程，所以此文专门针对性的解读`_YYModelMeta`实例化过程
# 解读

```Objective-C
_YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:cls];
```

上层NSObject的Category YYModel仅仅调用了一下api，接下来我们将这一行进行展开，详细看看里边到底做了什么。
```Objective-C
+ (instancetype)metaWithClass:(Class)cls {
    if (!cls) return nil;
    static CFMutableDictionaryRef cache;
    static dispatch_once_t onceToken;
    static dispatch_semaphore_t lock;
    dispatch_once(&onceToken, ^{
        cache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        lock = dispatch_semaphore_create(1);
    });
    dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
    _YYModelMeta *meta = CFDictionaryGetValue(cache, (__bridge const void *)(cls));
    dispatch_semaphore_signal(lock);
    if (!meta || meta->_classInfo.needUpdate) {
        meta = [[_YYModelMeta alloc] initWithClass:cls];
        if (meta) {
            dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
            CFDictionarySetValue(cache, (__bridge const void *)(cls), (__bridge const void *)(meta));
            dispatch_semaphore_signal(lock);
        }
    }
    return meta;
}
```
这一段代码首先单例化了一个Dictionary缓存，并且先取缓存中的`_YYModelMeta`，如果没有的话就实例化`_YYModelMeta`
```Objective-C
- (instancetype)initWithClass:(Class)cls {
    YYClassInfo *classInfo = [YYClassInfo classInfoWithClass:cls];
    if (!classInfo) return nil;
    self = [super init];

    // Get black list
    modelPropertyBlacklist
    // Get white list
    modelPropertyWhitelist
    // Get container property's generic class
    modelContainerPropertyGenericClass
    // Create all property metas.
    NSMutableDictionary *allPropertyMetas = [NSMutableDictionary new];
    YYClassInfo *curClassInfo = classInfo;
    while (curClassInfo && curClassInfo.superCls != nil) { // recursive parse super class, but ignore root class (NSObject/NSProxy)
        for (YYClassPropertyInfo *propertyInfo in curClassInfo.propertyInfos.allValues) {
            if (!propertyInfo.name) continue;
            if (blacklist && [blacklist containsObject:propertyInfo.name]) continue;
            if (whitelist && ![whitelist containsObject:propertyInfo.name]) continue;
            _YYModelPropertyMeta *meta = [_YYModelPropertyMeta metaWithClassInfo:classInfo
                                                                    propertyInfo:propertyInfo
                                                                         generic:genericMapper[propertyInfo.name]];
            if (!meta || !meta->_name) continue;
            if (!meta->_getter || !meta->_setter) continue;
            if (allPropertyMetas[meta->_name]) continue;
            allPropertyMetas[meta->_name] = meta;
        }
        curClassInfo = curClassInfo.superClassInfo;
    }
    if (allPropertyMetas.count) _allPropertyMetas = allPropertyMetas.allValues.copy;
    // create mapper

    return self;
}
```
这个实例化代码有点长，我删减了一下。其实主要算是做了4件事情
* 实例化了`YYClassInfo`
* 回调`YYModel`协议
* 根据上一步协议的回调结果blacklist、whitelist和集合泛型创建`_YYModelPropertyMeta`
* 创建自定义的key与`_YYModelPropertyMeta`的映射

首先我来来看一下YYClassInfo的实例化，这个过程挺复杂的，也是核心部分
***实例化`YYClassInfo`***
```Objective-C
+ (instancetype)classInfoWithClass:(Class)cls {
    if (!cls) return nil;
    static CFMutableDictionaryRef classCache;
    static CFMutableDictionaryRef metaCache;
    static dispatch_once_t onceToken;
    static dispatch_semaphore_t lock;
    dispatch_once(&onceToken, ^{
        classCache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        metaCache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        lock = dispatch_semaphore_create(1);
    });
    dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
    YYClassInfo *info = CFDictionaryGetValue(class_isMetaClass(cls) ? metaCache : classCache, (__bridge const void *)(cls));
    if (info && info->_needUpdate) {
        [info _update];
    }
    dispatch_semaphore_signal(lock);
    if (!info) {
        info = [[YYClassInfo alloc] initWithClass:cls];
        if (info) {
            dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
            CFDictionarySetValue(info.isMeta ? metaCache : classCache, (__bridge const void *)(cls), (__bridge const void *)(info));
            dispatch_semaphore_signal(lock);
        }
    }
    return info;
}
```
这个实例化与`_YYModelMeta`的实例化方法是一样的，接下来我们重点关注`[[YYClassInfo alloc] initWithClass:cls];`这个方法中调用了一个核心方法
```Objective-C
- (void)_update {
    _ivarInfos = nil;
    _methodInfos = nil;
    _propertyInfos = nil;

    Class cls = self.cls;
    unsigned int methodCount = 0;
    Method *methods = class_copyMethodList(cls, &methodCount);
    if (methods) {
        NSMutableDictionary *methodInfos = [NSMutableDictionary new];
        _methodInfos = methodInfos;
        for (unsigned int i = 0; i < methodCount; i++) {
            YYClassMethodInfo *info = [[YYClassMethodInfo alloc] initWithMethod:methods[i]];
            if (info.name) methodInfos[info.name] = info;
        }
        free(methods);
    }
    unsigned int propertyCount = 0;
    objc_property_t *properties = class_copyPropertyList(cls, &propertyCount);
    if (properties) {
        NSMutableDictionary *propertyInfos = [NSMutableDictionary new];
        _propertyInfos = propertyInfos;
        for (unsigned int i = 0; i < propertyCount; i++) {
            YYClassPropertyInfo *info = [[YYClassPropertyInfo alloc] initWithProperty:properties[i]];
            if (info.name) propertyInfos[info.name] = info;
        }
        free(properties);
    }

    unsigned int ivarCount = 0;
    Ivar *ivars = class_copyIvarList(cls, &ivarCount);
    if (ivars) {
        NSMutableDictionary *ivarInfos = [NSMutableDictionary new];
        _ivarInfos = ivarInfos;
        for (unsigned int i = 0; i < ivarCount; i++) {
            YYClassIvarInfo *info = [[YYClassIvarInfo alloc] initWithIvar:ivars[i]];
            if (info.name) ivarInfos[info.name] = info;
        }
        free(ivars);
    }

    if (!_ivarInfos) _ivarInfos = @{};
    if (!_methodInfos) _methodInfos = @{};
    if (!_propertyInfos) _propertyInfos = @{};

    _needUpdate = NO;
}
```
这个方法是通过runtime实现了对Model中属性、变量、方法提取，然后分别实例化了`YYClassPropertyInfo`、`YYClassIvarInfo`、`YYClassMethodInfo`这三个类，并且存储到了相应的数组之中
***根据上一步协议的回调结果blacklist、whitelist和集合泛型创建_YYModelPropertyMeta***
这个过程主要是以下核心代码
```Objective-C
_YYModelPropertyMeta *meta = [_YYModelPropertyMeta metaWithClassInfo:classInfo
                                                        propertyInfo:propertyInfo
                                                             generic:genericMapper[propertyInfo.name]];
```
这个代码内部其实就是根据`YYClassPropertyInfo`生成一个业务类，包括key的名字、类型、泛型类、是否是数字、setter方法、getter方法等等，其实主要是在`YYClassPropertyInfo`这个类的基础了增加了一些业务逻辑的数据
***创建自定义的key与_YYModelPropertyMeta的映射***
这个过程就是根据`YYModel`协议中的`modelCustomPropertyMapper`方法，将自定义的key的键值对的映射进行处理
```Objective-C
if ([mappedToKey isKindOfClass:[NSString class]]) {
  ...
} else if ([mappedToKey isKindOfClass:[NSArray class]]) {
  ...           
}
```
这个是两种情况，一种是字符串(两种情况一种是单个的，一种是带`.`，称之为keyPath)；一种是数组的情况(一个model属性对应多个不同的key)。这两种情况都有一个单向链表实现

通过以上对数据的处理`_YYModelMeta`中的属性_mapper、_allPropertyMetas、_keyPathPropertyMetas、_multiKeysPropertyMetas、_keyMappedCount等属性则均相应的存储上了数据
# 总结
这一层数据层设计的也相当的巧妙，有专门存储model信息的类，有专门处理我们自定义的业务逻辑类。上下分了两层。
