---
title: iOS关联对象 # 这是标题
date: 2018-12-28 09:20:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 背景
在iOS开发中如果我们想给一个对象动态添加属性或者给`category`添加属性的时候，都是通过runtime的关联对象去实现，那我们添加的属性到底是如何存取的呢？是直接添加到了对象自身的内存中了去吗？带着这些疑问让我们看一runtime的源码，解开关联对象的神秘面纱。
# 关联对象源码

## 存值
```Objective-C
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
    _object_set_associative_reference(object, (void *)key, value, policy);
}

```
我们调用此方法的时候，一共传递了四个参数：

| 参数名称 | 解释 |
| :------:| :------: |
|id object|需要关联的对象|
|void *key|对应的key|
|id value|对应的值|
|objc_AssociationPolicy policy|内存管理策略|
内存管理策略:
```C
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object.
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied.
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```
对于四个参数理解完了之后让我们看看它真正的实现函数`_object_set_associative_reference`
```C
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);//得到对象地址
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);//首先通过对象的地址获取对象的hashmap
            if (i != associations.end()) {//判断是否已经存在，已经存在
                // secondary table exists
                ObjectAssociationMap *refs = i->second;//取值，对应的map
                ObjectAssociationMap::iterator j = refs->find(key);//通过key查找
                if (j != refs->end()) {//如果已经存在
                    old_association = j->second;//取到原来老的值,以便后边对其释放
                    j->second = ObjcAssociation(policy, new_value);//存储新的值
                } else {//不存在
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {//如果不存在,创建一个
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {//不存在则创建一个
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
```
通过以上代码我们可以看出其实关联对象在存储的时候在，生成了一个`AssociationsManager`单例对象，所以应用中所有的管理对象都存储于此`AssociationsManager`中。

具体存储的实现是借助了C++的关联容器`unordered_map`实现的。具体可以参看代码中我加的注释。

整个过程就是通过`object`对象的地址存储了一个类似`hashmap`的东西;取到此hashmap，然后通过键值对的方式将我们需要存储的值存储到此`hashmap`中，这个过程中如果有旧值，则最后会将旧值就行释放
## 取值
取值的过程其实就比较简单了，就相当于从一个hashmap中取值
```c
id objc_getAssociatedObject(id object, const void *key) {
    return _object_get_associative_reference(object, (void *)key);
}
```

```Objective-C
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
                    objc_retain(value);
                }
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        objc_autorelease(value);
    }
    return value;
}
```
