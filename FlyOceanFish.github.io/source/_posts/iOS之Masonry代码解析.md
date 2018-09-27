---
title: iOS之Masonry代码解析 # 这是标题
date: 2018-9-21 9:20:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 背景
iOS平台上`Autolayout`布局对于屏幕的适配简直就是一把利剑，如果用xib或Storyboard进行布局对于每个人来说肯定是得心应手。不过如果对于纯代码布局的人来说，使用苹果苹果原生的api写过的人都知道是非常复杂的。所以`Masonry`横空出世，此框架主要是针对苹果`Autolayout`原生代码的封装，通过链式调用的方式，大大简化了使用复杂度，老少兼宜!
接下来就让我们分析一下它是如何封装的呢？
# 架构图
![](https://user-gold-cdn.xitu.io/2018/9/21/165f9effd66d0a4e?w=1567&h=825&f=png&s=118924)
首先让我们对于架构图做个简要说明
相对来说`Masonry`一共分为三层。
* `View+MASAdditions`:第一层.提供最外层make、update、remake三个主要方法创建约束；并且提供了获取约束的方法。
* `MASConstraintMaker`:第二层。提供一个桥梁的作用，也是核心作用。我们生产的所有约束均存在此类的一个数组里。当调用`install`方法的时候，会循环遍历数组
* `MASConstraint`:第三层。这个是两个子类的父类，是`Masonry`约束的载体，并且拥有一个`MASViewAttribute`类的对象，用于存储苹果`Autolayout`的载体和真正作用的视图。
>`MASCompositeConstraint`此子类主要是针对`make.width.height`这种链式调用，通过数组的形式存储了多个约束条件。也即多个`MASViewConstraint`此类对象。所以`MASCompositeConstraint`这个类的操作最终还是调用了`MASViewConstraint`这个类内的方法。
# 源码解析

```Objective-C
[redView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding);
}];
```
>注意：以上代码不是一个视图的完整约束。我们分析只要通过一个约束解析就好，代码处理逻辑都是一样的
这个代码我们调用的一个创建约束的最外层api，接下来让我们进去看看它里边的实现。

```Objective-C
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    block(constraintMaker);
    return [constraintMaker install];
}
```
这里实例化了一个constraintMaker供我们上层api的使用，通过调用block，执行了我们创建约束的代码，然后调用`install`方法，最终完成了视图约束的创建。

接下来让我分两步分析这里边的核心实现。

*  `make.top.equalTo(superview.mas_top).with.offset(padding);`这个代码的实现完成了约束对象的创建

* `[constraintMaker install]`完成了最终我们创建的约束对应作用于视图

## `make.top.equalTo(superview.mas_top).with.offset(padding);`解析

这个点语法，每次调用要不就是创建了一个约束，要不就是更改约束的值

我们首先来看`make.top`
```C
- (MASConstraint *)top {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}

- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}
```
这个方法比较简单，主要是对api的调用，并且传入了`NSLayoutAttributeTop`苹果层面约束的名称

接下来让我们看约束添加的核心方法，所有添加约束的方法都是调用此方法
```Objective-C
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        return compositeConstraint;
    }
    if (!constraint) {
        newConstraint.delegate = self;
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```
这个方法有两个分支，一个是传入nil,也是目前我们分析的情况，另外一种情况我们稍后展开分析，其实就是`make.width.height`这种连续的情况，此时不会出入nil的。
言归正传，当出入nil的时候，首先实例化了`viewAttribute`和`newConstraint`两个变量。`viewAttribute`此变量持有作用的视图和约束名称，`newConstraint`是`Masonry`层面的约束，实现约束的具体逻辑。
其次设置了delegate，并且加入了`constraints`数组中。这个数组存储了所有的约束对象。最后返回了`newConstraint`对象

`make.top`至此已经分析结束，接着我们看`make.top.equalTo(superview.mas_top)`。这个方法我们最应该学习的,这个方法设计的也比较巧妙。第一回看可能不知道这个语法是怎么回事呢。接下来让我们看看`MASConstraint`类中的`equalTo`方法的实现。
```Objective-C
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}
```
这个方法是返回了一个block，所以`.equalTo()`括号调用的就是block执行。block返回`MASConstraint *`一个对象。

接着让我们看看`MASViewConstraint`中`.offset()`方法

```Objective-C
- (void)setOffset:(CGFloat)offset {
    self.layoutConstant = offset;
}
```
这个方法比较简单，只是给`MASViewConstraint`对象设置了一个具体的值。

到此这个方法已经分析完成。

接下来让我们看看上边提到的另一种调用方式`make.width.height`

这种调用我们就不一一展开分析了，主要我们来看`.height`,调用这个方法的时候会执行上文说的`- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute`中以下代码
```Objective-C
if ([constraint isKindOfClass:MASViewConstraint.class]) {
    //replace with composite constraint
    NSArray *children = @[constraint, newConstraint];
    MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
    compositeConstraint.delegate = self;
    [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
    return compositeConstraint;
}
```
这里是实例化了`MASConstraint`另外一个子类`MASCompositeConstraint`，这个类其实很简单，就是用来存储多个`MASViewConstraint`对象，然后循环遍历操作每个对象


## `[constraintMaker install]`解析

```Objective-C
- (NSArray *)install {
    if (self.removeExisting) {
        NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
        for (MASConstraint *constraint in installedConstraints) {
            [constraint uninstall];
        }
    }
    NSArray *constraints = self.constraints.copy;
    for (MASConstraint *constraint in constraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
    [self.constraints removeAllObjects];
    return constraints;
}
```
这个方法就是一个循环遍历的操作。不过`[constraint install];`这个方法会调用`MASViewConstraint`的install和`MASCompositeConstraint`的install。
### `MASViewConstraint`的install
```Objective-C
- (void)install {
    if (self.hasBeenInstalled) {
        return;
    }

    if ([self supportsActiveProperty] && self.layoutConstraint) {
        self.layoutConstraint.active = YES;
        [self.firstViewAttribute.view.mas_installedConstraints addObject:self];
        return;
    }

    MAS_VIEW *firstLayoutItem = self.firstViewAttribute.item;
    NSLayoutAttribute firstLayoutAttribute = self.firstViewAttribute.layoutAttribute;
    MAS_VIEW *secondLayoutItem = self.secondViewAttribute.item;
    NSLayoutAttribute secondLayoutAttribute = self.secondViewAttribute.layoutAttribute;

    // alignment attributes must have a secondViewAttribute
    // therefore we assume that is refering to superview
    // eg make.left.equalTo(@10)
    if (!self.firstViewAttribute.isSizeAttribute && !self.secondViewAttribute) {
        secondLayoutItem = self.firstViewAttribute.view.superview;
        secondLayoutAttribute = firstLayoutAttribute;
    }

    MASLayoutConstraint *layoutConstraint
        = [MASLayoutConstraint constraintWithItem:firstLayoutItem
                                        attribute:firstLayoutAttribute
                                        relatedBy:self.layoutRelation
                                           toItem:secondLayoutItem
                                        attribute:secondLayoutAttribute
                                       multiplier:self.layoutMultiplier
                                         constant:self.layoutConstant];

    layoutConstraint.priority = self.layoutPriority;
    layoutConstraint.mas_key = self.mas_key;

    if (self.secondViewAttribute.view) {
        MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
        NSAssert(closestCommonSuperview,
                 @"couldn't find a common superview for %@ and %@",
                 self.firstViewAttribute.view, self.secondViewAttribute.view);
        self.installedView = closestCommonSuperview;
    } else if (self.firstViewAttribute.isSizeAttribute) {
        self.installedView = self.firstViewAttribute.view;
    } else {
        self.installedView = self.firstViewAttribute.view.superview;
    }


    MASLayoutConstraint *existingConstraint = nil;
    if (self.updateExisting) {
        existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
    }
    if (existingConstraint) {
        // just update the constant
        existingConstraint.constant = layoutConstraint.constant;
        self.layoutConstraint = existingConstraint;
    } else {
        [self.installedView addConstraint:layoutConstraint];
        self.layoutConstraint = layoutConstraint;
        [firstLayoutItem.mas_installedConstraints addObject:self];
    }
}
```
这个方法则是针对maker的2种情况调用了苹果底层api，进行约束的创建。分别是更新约束和重新创建约束。
### `MASCompositeConstraint`的install
```Objective-C
- (void)install {
    for (MASConstraint *constraint in self.childConstraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
}
```
此方法就是将存储的`MASViewConstraint`或`MASCompositeConstraint`对象，循环遍历调用上述的intall方法。如果是`MASCompositeConstraint`对象对象，则会递归调用。
# 总结
Masonry整个架构还是比较简单的，值的我们学习的一方面是它的整个架构设计，另一方面很值得我们学习的是在OC中如何实现链式调用
