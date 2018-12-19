---
title: 开发笔记-警惕swizzling
date: 2018-09-14 12:00:00
tags: 开发笔记
---

不知道什么时候开始，只要使用了`swizzling`都能被解读成是`AOP`开发，开发者张口嘴就是`runtime`，将其高高捧起，称之为`黑魔法`；以项目中各种`method_swizzling`为荣，却不知道这种做法破坏了代码的整体性，使关键逻辑支离破碎。本文基于[iOS界的毒瘤](https://www.valiantcat.cn/index.php/2017/11/03/53.html#menu_index_7)一文，从另外的角度谈谈为什么我们应当`警惕`

## 调用顺序性
`调用顺序性`是链接文章讲述的的核心问题，它会破坏方法的原有执行顺序，导致意料之外的错误。先从一段简单的代码聊起：

    @interface SLTestObject: NSObject
    
    @end
    
    @implementation SLTestObject
    
    - (instancetype)init {
        self = [super init];
        return self;
    }

    @end
    
    void testIsSelectorSame() {
        Method allocate1 = class_getClassMethod([NSObject class], @selector(alloc));
        Method allocate2 = class_getClassMethod([SLTestObject class], @selector(alloc));
        
        Method initialize1 = class_getInstanceMethod([NSObject class], @selector(init));
        Method initialize2 = class_getInstanceMethod([SLTestObject class], @selector(init));
        
        assert(allocate1 == allocate2 && initialize1 != initialize2);
    }
    
这段代码的目的是证明一个定论：

> 如果子类没有重写父类声明的方法，在子类对象调用该方法时，执行的是父类实现的代码

基于这一定论，假定一个场景：现在通过无埋点方案统计用户进入和离开`Controller`次数：

    @implementation UIViewController (SLCount)
    
    + (void)load {
        sl_swizzle([self class], @selector(viewWillAppear:), @selector(sl_viewWillAppearI:));
        sl_swizzle([self class], @selector(viewDidDisappear:), @selector(sl_viewDidDisappearI:));
    }
    
    - (void)sl_viewWillAppearI: (BOOL)animated {
        [SLControllerCounter countControllerEnter: [self class]];
        [self sl_viewWillAppearI: animated];
    }
    
    - (void)sl_viewDidDisappearI: (BOOL)animated {
        [SLControllerCounter countControllerLeave: [self class]];
        [self sl_viewDidDisappearI: animated];
    }
    
    @end
    
由于`UIViewController`是所有控制器的父类，所以理论上只要`swizzle`这个类就能统计到所有控制器的信息。同时项目中存在一个定制的基础控制器`SLBaseViewController`存在这么一段代码：

    @implementation SLBaseViewController (SLCount)
    
    + (void)load {
        sl_swizzle([self class], @selector(viewWillAppear:), @selector(sl_viewWillAppearII:));
        sl_swizzle([self class], @selector(viewDidDisappear:), @selector(sl_viewDidDisappearII:));
    }
    
    - (void)sl_viewWillAppearII: (BOOL)animated {
        [self prepareRequest];
        [self sl_viewWillAppearII: animated];
    }
    
    - (void)sl_viewDidDisappearII: (BOOL)animated {
        [self sl_viewDidDisappearII: animated];
        [self cancelAllRequests];
    }
    
    @end

但是这两段代码却在特定的场景下发生`crash`，发生异常的原因在于子类在没有重写方法的情况下，子类先于父类进行了`swizzle`的操作。`iOS`使用中方法名称`SEL`和方法实现`IMP`是分开存放的，使用结构体`Method`将两者关联到一起：

    typedef struct Method {
        SEL name;
        IMP imp;
    } Method;
    
交换方法会将两个`method`中的`imp`进行交换。而在理想情况下，父类先于子类完成了`swizzle`，原有方法保存了`swizzle`之后的`imp`，这时候子类再进行`swizzle`就能正确调用。下图标识了`SEL`和`IMP`的关联，箭头表示`IMP`的调用次序：

![](https://upload-images.jianshu.io/upload_images/783864-ea12e2b4ac1dcbaf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是如果子类的`swizzle`发生的更早，这时候`viewWillAppear`对应的`imp`已经被修改，父类再进行`swizzle`的时候，调用次序已经出错：

![](https://upload-images.jianshu.io/upload_images/783864-256bd9dc6ed4626c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决方式也并不复杂，包括：

1. 在`swizzle`之前先`addMethod`，保证子类不沿用父类的默认实现
2. 每次调用通过`sel`去获取`imp`执行

具体的实现代码可以参考[iOS界的毒瘤](https://www.valiantcat.cn/index.php/2017/11/03/53.html#menu_index_7)的解决方案

## 行为冲突
在`OOP`的设计中，将描述对象抽象成类，将对象行为抽象成接口。从工程师的角度来说，职责单一的接口更利于迭代维护。类一旦设计好，应当不改动或者少改动接口。对于设计良好的接口来说，`swizzle`很可能直接破坏了整个接口的行为：

![](https://upload-images.jianshu.io/upload_images/783864-6cb67e06408cf585.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

举个例子，`crash防护`是当下被追捧的工具，但其中`KVO`的防护或许是一种很烂的手段。从实现来说，为了避免`KVO`导致的循环引用，需要在引用关系的中间插入一个`weakProxy`来做防护，因此监听代码实际上可以转换成：

    // 表面代码
    [observedObj addObserver: self forKeyPath: keyPath options: NSKeyValueObservingOptionNew context: nil];
    
    // 实际效果
    WeakProxy *proxy = [WeakProxy new];
    proxy.client = self;
    [observedObj addObserver: proxy forKeyPath: keyPath options: NSKeyValueObservingOptionNew context: nil];
    
为什么说这种设计很烂的？一旦客户端出现这样的代码：

    - (void)dealloc {
        ......
        [observedObj removeObserver: self forKeyPath: keyPath];
    }
    
通常情况下，以现在的多数`防护工具`的实现，都会发生崩溃。对于`swizzle`代码外的使用者来说，或许根本不清楚`observer`早已发生了转移，导致了原有的正确调用出错。解决方案之一是对`remove`接口同样进行`swizzle`，使得两次调用的监听对象配套：

    - (void)sl_removeObserver: (id)observer forKeyPath: (NSString *)keyPath {
        [self sl_removeObserver: observer.proxy forKeyPath: keyPath];
    }
    
然而这样做之后，首先`KVO`的行为已经被修改，接口被破坏可能导致潜在的隐患。其次，如果存在多个防护工具，如果按照`weakProxy`的实现，那么一旦有`2`个或者更多的防护时，`KVO`功能将失效：

    OneWeakProxy *proxy = [OneWeakProxy new];
    proxy.client = self;    
    [observedObj addObserver: proxy forKeyPath: keyPath options: NSKeyValueObservingOptionNew context: nil];
    
    TwoWeakProxy *proxy = [TwoWeakProxy new];
    proxy.client = self;    /// self is OneWeakProxy
    [observedObj addObserver: proxy forKeyPath: keyPath options: NSKeyValueObservingOptionNew context: nil];

在第二次生成`WeakProxy`后并调用方法后，`OneWeakProxy`创建的对象被释放。如果要避免多个防护工具对流程造成干扰，还需要做更多额外的工作。况且一旦有其中一个没有完美实现，整套`防护机制`可能就直接崩溃失效了，因此`KVO防护`不见得是一种好手段

![](https://upload-images.jianshu.io/upload_images/783864-2286ae22b41ecbd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 代码整体性
以上面例子来说，`KVO`是`NSObject`这个基类提供的能力，由于`子类默认沿用父类的方法实现`这一原则，这种方法的`swizzle`实际上影响了全部的对象，例如下面的代码实际上效果是完全一样的：

    /// swizzle 1
    void swizzleTableView() {
        Method ori = class_getClassMethod([UITableView class], @selector(addObserver:forKeyPath:options:context:));
        Method cus = class_getClassMethod([UITableView class], @selector(sl_addObserver:forKeyPath:options:context:));
        method_exchange(ori, cus);
    }
    
    /// swizzle 2
    void swizzleObj() {
        Method ori = class_getClassMethod([NSObject class], @selector(addObserver:forKeyPath:options:context:));
        Method cus = class_getClassMethod([NSObject class], @selector(sl_addObserver:forKeyPath:options:context:));
        method_exchange(ori, cus);
    }
    
而第一个方法由于默认实现是`NSObject`的，因此一旦发生了`swizzle`所有的对象都会生效，这存在两个问题：

1. 非`UITableView`对象依旧受到了`KVO`的拦截影响
2. 没有`sl_addObserver:forKeyPath:options:context:`的对象会发生崩溃

另一方面，类的接口设计总是偏向于`装扮模式`的思维，不同层级的类对象在自己的方法被调用起时会执行自身特有的工作，这种设计让继承有足够的灵活性，从`viewDidLoad`的实现代码可见一斑：

    - (void)viewDidLoad {
        [super viewDidLoad];
        /// setup work
    }

换句话说，以这种`装扮模式`思维来构建的代码，如果中间的一个方法被影响甚至破坏了，在中间的这个类开始往下将呈现塌式破坏，可以想象如果`UIView`一旦出错，应用几乎丧失展示控件的能力。但假如确实需要`swizzle`的中间环节，必须保证`swizzle`不对或者尽量少地对子类对象造成影响

![关注我的公众号获取更新信息](https://upload-images.jianshu.io/upload_images/783864-5f15782c42a970c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


