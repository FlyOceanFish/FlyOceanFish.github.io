---
title: 读YYCache源码总结 # 这是标题
date: 2018-04-02 11:07:00
updated: 2018-04-02 11:07:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 开头
俗话说三人行必有我师，所以我们得多向他人学习。于是选了`YYCache`这一大神开源的缓存框架进行代码的研究和学习。作者这里仅仅做一个自身的总结和将一些感悟写出了。

# 思路简析
`YYCache`包含两部分：内存缓存(`YYMemoryCache`)和物理缓存(`YYDiskCache`)。通过`YYCache`对两者的封装，提供统一接口，完成了对缓存的实现。
## 存数据
```Objective-C
- (void)setObject:(id<NSCoding>)object forKey:(NSString *)key {
    [_memoryCache setObject:object forKey:key];
    [_diskCache setObject:object forKey:key];
}
```
首先是将数据存入缓存，同时在物理硬盘上缓存一份
## 取数据
```Objective-C
- (id<NSCoding>)objectForKey:(NSString *)key {
    id<NSCoding> object = [_memoryCache objectForKey:key];
    if (!object) {
        object = [_diskCache objectForKey:key];
        if (object) {
            [_memoryCache setObject:object forKey:key];
        }
    }
    return object;
}
```
取数据首先是从内存取(因为内存中读取是比物理硬盘读取速度快很多的)，如果取不到(由于`YYMemoryCache`对可存大小、时间等的限制，会清除一些内存数据)；
这时候从物理硬盘取(其实物理硬盘也是有限制的，所以也不一定能取到),如果取到了***同时存入内存中***(这样下次取的时候，可以直接从内存中取出，速度会大大提升)

>其实大部分的的缓存实现比如`SDImageCache`等实现思路都是这样的，真正的差别主要是内存缓存和物理硬盘缓存具体实现细节上的差别

# `YYMemoryCache`
`YYMemoryCache`采用的是双向链表和Dictionary结合的技术，实现了增删时间复杂度是O(1)。双向链表通过MRU和LRU的思想进行删除和增加
```Objective-C
- (void)setObject:(id)object forKey:(id)key withCost:(NSUInteger)cost {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
    pthread_mutex_lock(&_lock);//使用pthread互斥锁，原先是OSSpinLock，后来作者更换了。不过可能由于后期好久没维护了，所以没使用苹果的os_unfair_lock替换品
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));//之前存储过的话就取出来并将其放到双向链表的第一个。MRU
    NSTimeInterval now = CACurrentMediaTime();
    if (node) {
        _lru->_totalCost -= node->_cost;
        _lru->_totalCost += cost;
        node->_cost = cost;
        node->_time = now;
        node->_value = object;
        [_lru bringNodeToHead:node];//放到第一个
    } else {
        node = [_YYLinkedMapNode new];//如果之前没有存储的话，实例化一个并且插入头部
        node->_cost = cost;
        node->_time = now;
        node->_key = key;
        node->_value = object;
        [_lru insertNodeAtHead:node];
    }
    if (_lru->_totalCost > _costLimit) {
        dispatch_async(_queue, ^{
            [self trimToCost:_costLimit];
        });
    }
    if (_lru->_totalCount > _countLimit) {
        _YYLinkedMapNode *node = [_lru removeTailNode];//如果超过数量限制，删除链表的最后一个。LRU
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                [node class]; //随便调一下方法，目的就是将对象的释放放到子线程里边
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }
    pthread_mutex_unlock(&_lock);
}
```

# `YYDiskCache`
`YYDiskCache`物理缓存采用的是文件和SQLite结合的方式，作者是低于20k存储到SQLite，超过20k存储文件。关于20k的阈值，本人有点疑惑。 [SQLite 官网的描述](http://www.sqlite.org/intern-v-extern-blob.html)官网最后的结论写的是100k，20k有可能YYCache的作者是通过观察实现数据做出的结论吧。
物理缓存使用的锁是信号，主要是信号不用消耗CPU
```Objective-C
- (void)setObject:(id<NSCoding>)object forKey:(NSString *)key {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }

    NSData *extendedData = [YYDiskCache getExtendedDataFromObject:object];
    NSData *value = nil;
    if (_customArchiveBlock) {
        value = _customArchiveBlock(object);
    } else {
        @try {
            value = [NSKeyedArchiver archivedDataWithRootObject:object];
        }
        @catch (NSException *exception) {
            // nothing to do...
        }
    }
    if (!value) return;
    NSString *filename = nil;
    if (_kv.type != YYKVStorageTypeSQLite) {//YYCache默认使用的是混合模式
        if (value.length > _inlineThreshold) {//是否大于阈值20k，大于的话使用文件存储
            filename = [self _filenameForKey:key];
        }
    }

    Lock();//信号锁
    [_kv saveItemWithKey:key value:value filename:filename extendedData:extendedData];
    Unlock();
}
```
# 感悟
本文没有很详细的解析源码，主要提取了一些重要片段进行了解析。因为YYCache的整体源码思路非常清晰，代码很整洁，所以读起来一点不费事，这里主要讲述思路，详细的完整实现建议亲自去读源码。非常佩服YYKit大神的工匠精神，为了写一个YYCache框架其实是调研了许多的第三方框架和资料。
