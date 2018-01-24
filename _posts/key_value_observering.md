---
title: 实现KVO
date: 2018-01-21 08:00:00
categories:
- Note
tags:
- Note
- Tips
- Design
---

基于观察者设计模式，苹果实现了`notification`和`kvo`两套监听机制，两者都实现了`一对多`的监听支持。通知在设计上暴露了`notificationCenter`这个中心类，通过公开的接口和数据类型，不难猜测出其实现方式。但`KVO`仅在`NSObject`中暴露了几个接口，同时缺乏必要的中间类，文档中也只有模糊的介绍，这让人不由地对其实现机制产生兴趣。

> Automatic key-value observing is implemented using a technique called isa-swizzling... When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class ..

翻译过来就是：`KVO`是通过一种称作`isa-swizzling`的机制实现的，这个机制会在被观察对象的属性被监听时修改对象的`isa`指针，让指针指向一个中间类而非对象自身的类。

## isa
如果你还能用英语而非拼音的方式命名变量，那么`Objective-C`你可以很容易就看出这是基于`C`语言封装的面向对象语言，这也意味着复杂的`class`结构只是基于`struct`实现的。抛开这个封装的难度不谈，

[](https://opensource.apple.com/source/objc4/objc4-532/runtime/runtime.h)

## 从观察者说起
从高中开始，哥们几个总会在教室的电脑里装上一款《拳皇98》。在晚饭过后、晚修之间切磋几把，岂不快哉。但总有刁民想害朕，班主任是这一优良传统的破坏者，为了避免这唯一的休闲活动受到打搅，总有一个小伙伴兼职放风的职责，观察班主任是否过来了。

![](http://p0zs066q3.bkt.clouddn.com/2018012801.png)

在这个过程中，总共存在三种角色：打游戏的`监听者`、放风的`观察者`以及班主任`被观察对象`，这三种角色共同构建了`观察者模式`。


## 扩展阅读
[从NSObject的初始化了解isa](https://draveness.me/isa)

[神经病院Objective-C Runtime入院第一天](https://halfrost.com/objc_runtime_isa_class/)

[神经病院Objective-C Runtime住院第二天](https://halfrost.com/objc_runtime_objc_msgsend/)

[神经病院Objective-C Runtime出院第三天](https://halfrost.com/how_to_use_runtime/)


