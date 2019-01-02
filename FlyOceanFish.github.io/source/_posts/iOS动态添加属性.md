---
title: iOS动态添加属性 # 这是标题
date: 2019-1-2 13:45:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 背景
之前一篇文章《iOS关联对象》详细介绍了如何通过关联对象添加属性，本篇文章将介绍如何通过runtime的`class_addProperty`或`class_addIvar`动态添加属性，并且带领大家看看这两个方法底层是如何实现的。
# `class_addProperty`添加属性
对于已经存在的类我们用`class_addProperty`方法来添加属性，而对于动态创建的类我们通过`class_addIvar`添加属性，
它会改变一个已有类的内存布局，一般是通过`objc_allocateClassPair`动态创建一个class，才能调用`class_addIvar`创建Ivar，最后通过`objc_registerClassPair`注册class。

>对于已经存在的类，`class_addIvar`是不能够添加属性的

首先我们声明了一个`Person`类
```Objective-C
@interface Person : NSObject
@property (nonatomic,copy)NSString *name;
@end
```
然后我们通过`class_addProperty`动态添加属性
```Objective-C
id getter(id object,SEL _cmd1){
    NSString *key = NSStringFromSelector(_cmd1);
    return objc_getAssociatedObject(object, (__bridge const void * _Nonnull)(key));
}
void setter(id object,SEL _cmd1,id newValue){
    NSString *key = NSStringFromSelector(_cmd1);
    key = [[key substringWithRange:NSMakeRange(3, key.length-4)] lowercaseString];
    objc_setAssociatedObject(object, (__bridge const void * _Nonnull)(key), newValue, OBJC_ASSOCIATION_RETAIN);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        objc_property_attribute_t type = { "T", [[NSString stringWithFormat:@"@\"%@\"",NSStringFromClass([NSString class])] UTF8String] }; //type
        objc_property_attribute_t ownership0 = { "C", "" }; // C = copy
        objc_property_attribute_t ownership = { "N", "" }; //N = nonatomic
        objc_property_attribute_t backingivar  = { "V", [[NSString stringWithFormat:@"_%@", @"sex"] UTF8String] };  //variable name
        objc_property_attribute_t attrs[] = { type, ownership0, ownership,backingivar};//这个数组一定要按照此顺序才行
        BOOL add = class_addProperty([Person class], "sex", attrs, 4);
        if (add) {
            NSLog(@"添加成功\n");
        }else{
            NSLog(@"添加失败\n");
        }

        class_addMethod([Person class], NSSelectorFromString(@"sex"), (IMP)getter, "@@:");
        class_addMethod([Person class], NSSelectorFromString(@"setSex:"), (IMP)setter, "v@:@");

        unsigned int count;
        objc_property_t *properties =class_copyPropertyList([Person class], &count);
        for (int i = 0; i < count; i++) {
            objc_property_t property = properties[i];
            NSLog(@"名字:%s--属性:%s\n",property_getName(property),property_getAttributes(property));
        }

        Person *person = [Person new];
        person.name = @"FlyOceanFish";
        [person setValue:@"男" forKey:@"sex"];
        NSLog(@"name:%@",person.name);
        NSLog(@"sex:%@",[person valueForKey:@"sex"]);
    }
    return 0;
}
```

[Demo地址](https://github.com/FlyOceanFish/AddProperty)

这里要注意几点:
* attrs属性设置的数组一定是此顺序
此属性设置是参照原有属性的打印格式进行设置的，`name`是原先有的，`sex`是后来动态添加的,如下：
>2019-01-02 14:39:39.370405+0800 MyProperty[1354:159207] 名字:sex--属性:T@"NSString",C,N,V_sex
>
>2019-01-02 14:39:39.370425+0800 MyProperty[1354:159207] 名字:name--属性:T@"NSString",C,N,V_name
* 添加完属性我们要添加上对应的set、get方法，因为我们是通过kvo的方式设值和取值的，它会调用set、get方法，如果没有的话，会报错。

# 底层代码实现
上边代码演示了如何动态添加属性，接下来让我们看看苹果底层是如何实现的。
## class_addProperty
```Objective-C
BOOL
class_addProperty(Class cls, const char *name,
                  const objc_property_attribute_t *attrs, unsigned int n)
{
    return _class_addProperty(cls, name, attrs, n, NO);
}
```

```C
static bool
_class_addProperty(Class cls, const char *name,
                   const objc_property_attribute_t *attrs, unsigned int count,
                   bool replace)
{
    if (!cls) return NO;
    if (!name) return NO;

    property_t *prop = class_getProperty(cls, name);
    if (prop  &&  !replace) {
        // already exists, refuse to replace
        return NO;
    }
    else if (prop) {
        // replace existing
        rwlock_writer_t lock(runtimeLock);
        try_free(prop->attributes);
        prop->attributes = copyPropertyAttributeString(attrs, count);
        return YES;
    }
    else {
        rwlock_writer_t lock(runtimeLock);

        assert(cls->isRealized());

        property_list_t *proplist = (property_list_t *)
            malloc(sizeof(*proplist));
        proplist->count = 1;
        proplist->entsizeAndFlags = sizeof(proplist->first);
        proplist->first.name = strdupIfMutable(name);
        proplist->first.attributes = copyPropertyAttributeString(attrs, count);

        cls->data()->properties.attachLists(&proplist, 1);

        return YES;
    }
}
```
通过代码我们可以看到如果添加一个已经存在的属性是添加不成功的；
添加一个新属性，是实例化了一个`property_list_t`对象，最终调用了cls的data方法,返回了`class_rw_t`指针,最终添加在属性`properties`的一个数组中。
>还有一个结构体的名字是`class_ro_t`，与`class_rw_t`是配合使用的，大家有兴趣可以自行去研究。ro即read only;rw即read write。看到这里应该能猜个八九不离十

`class`对象结构体

```C
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() {
        return bits.data();
    }
    ...
  }
```
`class_rw_t`结构体
```C
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
    ...
  }
```
## class_addIvar
```C
BOOL

{
    if (!cls) return NO;

    if (!type) type = "";
    if (name  &&  0 == strcmp(name, "")) name = nil;

    rwlock_writer_t lock(runtimeLock);

    assert(cls->isRealized());

    // No class variables
    if (cls->isMetaClass()) {
        return NO;
    }

    // Can only add ivars to in-construction classes.
    if (!(cls->data()->flags & RW_CONSTRUCTING)) {
        return NO;
    }

    // Check for existing ivar with this name, unless it's anonymous.
    // Check for too-big ivar.
    // fixme check for superclass ivar too?
    if ((name  &&  getIvar(cls, name))  ||  size > UINT32_MAX) {
        return NO;
    }

    class_ro_t *ro_w = make_ro_writeable(cls->data());

    // fixme allocate less memory here

    ivar_list_t *oldlist, *newlist;
    if ((oldlist = (ivar_list_t *)cls->data()->ro->ivars)) {
        size_t oldsize = oldlist->byteSize();
        newlist = (ivar_list_t *)calloc(oldsize + oldlist->entsize(), 1);
        memcpy(newlist, oldlist, oldsize);
        free(oldlist);
    } else {
        newlist = (ivar_list_t *)calloc(sizeof(ivar_list_t), 1);
        newlist->entsizeAndFlags = (uint32_t)sizeof(ivar_t);
    }

    uint32_t offset = cls->unalignedInstanceSize();
    uint32_t alignMask = (1<<alignment)-1;
    offset = (offset + alignMask) & ~alignMask;

    ivar_t& ivar = newlist->get(newlist->count++);
#if __x86_64__
    // Deliberately over-allocate the ivar offset variable.
    // Use calloc() to clear all 64 bits. See the note in struct ivar_t.
    ivar.offset = (int32_t *)(int64_t *)calloc(sizeof(int64_t), 1);
#else
    ivar.offset = (int32_t *)malloc(sizeof(int32_t));
#endif
    *ivar.offset = offset;
    ivar.name = name ? strdupIfMutable(name) : nil;
    ivar.type = strdupIfMutable(type);
    ivar.alignment_raw = alignment;
    ivar.size = (uint32_t)size;

    ro_w->ivars = newlist;
    cls->setInstanceSize((uint32_t)(offset + size));

    // Ivar layout updated in registerClass.

    return YES;
}
```
通过以上代码我们可以看到通过此方法添加的属性是实例化了ivar_t对象，并且存储在了`class_ro_t`对象中了。所以跟我们上边说的改变了类的内存布局一致。
# 总结
我们通过一个demo实现了动态添加属性，通过底层源码解析让大家彻底认识了`class_addProperty`和`class_addIvar`两个方法。runtime让oc成为了一门动态语言，只有我们想不到的，没有runtime做不到的。
