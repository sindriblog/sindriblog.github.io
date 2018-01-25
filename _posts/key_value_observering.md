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
通过文档的描述，可以得出`isa`指针是`KVO`的实现机制中最为核心的变量，那么什么是`isa`指针？如果你能使用英语而非拼音来书写代码，那么一定能够明白`Objective-C`翻译过来就是`C`语言的面向对象。换句话说：

> OC的所有对象都是封装于C语言的结构体

虽然可以想象到，使用`struct`来实现面向对象的特性必然是一个十分复杂的过程，但继承的实现我们可以轻易的想象出来：在自身结构内部预留父结构体的变量。打个比方，`NSObject`的结构体为`objc_object`，存储了一个`isa`指针，假如存在子类`Person`，翻阅[objc-private](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-private.h.auto.html)可以确定子类的结构组成：

    struct objc_object {
        isa_t isa;
    };
    
    struct Person {
        struct objc_object {
            isa_t isa;
        };
        ......
    }
    
由于`NSObject`是所有类的父类，因此可以确定的是，每个对象的结构中都有一个`isa`指针。那么`isa`用来干嘛？通过`objc_object`的结构得出这个指针的类型是`isa_t`，在去掉不需要的编译器匹配代码后其结构如下：

    union isa_t 
    {
        isa_t() { }
        isa_t(uintptr_t value) : bits(value) { }
    
        Class cls;
        uintptr_t bits;
        
        ......
    };
    
这里面最重要的一个变量是`cls`，表示的是这个结构体代表的是哪个类。可以得出结论，`isa`指针决定了对象的所属类型。除此之外`isa`还存储了很多额外信息，如果你想了解更多，可以通过文末的扩展阅读了解更多。

### isa-swizzling
顾名思义，`isa-swizzling`是用来修改`isa`指针的机制，通过源码我们已经知道`isa`用来表示对象的所属类型，那么交换`isa`指针可以看做是修改对象的所属类型。正常情况下，由于`isa`指针存在私有文件中，根本不对外暴露，开发者是没有机会修改这个指针的，但是`runtime.h`中暴露了`object_setClass`函数，它允许我们修改一个对象的`class`，等同于修改了这个对象的`isa`指针。

    /// NSObject.mm
    - (Class)class {
        return object_getClass(self);
    }

    /// code
    id obj = [NSObject new];
    NSLog(@"-class: %@, object_getClass: %@", NSStringFromClass([obj class]), NSStringFromClass(object_getClass(obj)));
    
    object_setClass(obj, [NSString class]);
    NSLog(@"-class: %@, object_getClass: %@", NSStringFromClass([obj class]), NSStringFromClass(object_getClass(obj)));
    
    /// log
    2018-01-25 09:58:46.870577+0800 ThreadTest[11398:955919] -class: NSObject, object_getClass: NSObject
    2018-01-25 09:58:46.870743+0800 ThreadTest[11398:955919] -class: NSString, object_getClass: NSString
    
首先可以看到在`class`的实现中采用`object_getclass`函数获取对象的所属类型，如果`class`方法被重写导致方法没有返回实际类型，那就可以直接调用`object_getclass`来获取正确的对象类型。其次，上述代码中通过`objc_setClass`修改对象后，调用`class`或者`object_getclass`返回的结果都是`NSString`，这证明了`isa`指针已经被成功修改。但这种做法是存在风险的，假如被替换的类型和新类型的内存布局不是一致的情况下，极有可能造成意料之外的`bug`。

### 内存布局
由于`class`最终会被转换成`struct`，其次`class`的结构相对复杂，因此接下来通过`struct`来聊聊内存布局。虽然`struct`看起来塞进了许多乱七八糟的变量，但它的内存布局是有迹可循的，结构体的内存对齐必须满足这两个条件：

1. 整个`struct`的地址必须是最大变量地址长度的正整数倍
2. 成员变量`x`在`struct`内的偏移量总是自身长度的整数倍

首先，背书大法好啊：许多计算机为了简化处理器和存储器系统之间的硬件设计，要求某种类型的数据地址必须是值`k`的倍数，通常`k`在`2、4、8`之间取值，如果数据长度符合`k`的倍数，那么硬件之间存取的效率会得到优化。以下的结构体在`32`位系统下为例：

    struct numSct {
        int a;
        long b;
        bool c;
    }
    
实际`numSct`需要的内存大小为`4 + 8 + 1 = 13`，但如果按照实际需要分配内存，读取数据存在两次不同取值方式的切换，这种开销在`struct`结构庞大时更为明显。因此编译器会以最大变量的占用长度作为`k`值，分配`8 + 8 + 8 = 24`个字节：

![](http://p0zs066q3.bkt.clouddn.com/2018012803.jpg)

考虑到这种内存布局可能会造成大量的内存损耗，`C`语言允许变量`x`前存在多个总长度不大于`x`的变量共同使用同一块区域。由于可能有些拗口，我们将`numSct`的结构更改一下：

    /// numSct1
    struct numSct {
        bool a;
        int b;
        long c;
    }
    
    /// numSct2
    struct numSct {
        int b;
        bool a;
        long c;
    }
    
通过更换变量的位置之后，虽然还是按照最大长度为`k`分配内存，但是`int`和`bool`类型的长度加起来小于`k`，因此可以共享一个`k`大小的空间：

![](http://p0zs066q3.bkt.clouddn.com/2018012802.jpg)

在这种情况下，可以把`bool`和`int`当做是另一个集合，这个集合会以最大长度变量即`int`的长度为准划分内存占用，此时可以看到变量`a`、`b`在结构体内的偏移量符合自身长度的整数倍。

### swizzling风险
通过`struct`的内存布局不难发现，如果类型在发生改变之后，新类型的内存布局和原来的内存布局不一致的时候，极有可能出现`crash`。

## 从观察者说起
从高中开始，哥们几个总会在教室的电脑里装上一款《拳皇98》。在晚饭过后、晚修之间切磋几把，岂不快哉。但总有刁民想害朕，班主任是这一优良传统的破坏者，为了避免这唯一的休闲活动受到打搅，总有一个小伙伴兼职放风的职责，观察班主任是否过来了。

![](http://p0zs066q3.bkt.clouddn.com/2018012801.png)

在这个过程中，总共存在三种角色：打游戏的`监听者`、放风的`观察者`以及班主任`被观察对象`，这三种角色共同构建了`观察者模式`。


## 扩展阅读
[从NSObject的初始化了解isa](https://draveness.me/isa)

[神经病院Objective-C Runtime入院第一天](https://halfrost.com/objc_runtime_isa_class/)

[神经病院Objective-C Runtime住院第二天](https://halfrost.com/objc_runtime_objc_msgsend/)

[神经病院Objective-C Runtime出院第三天](https://halfrost.com/how_to_use_runtime/)


