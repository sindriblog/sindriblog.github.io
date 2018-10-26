---
title: 质量监控-野指针定位
date: 2017-11-01 08:00:00
tags:
- 质量监控
---


> 当所指向的对象被释放或者收回，但是对该指针没有作任何的修改，以至于该指针仍旧指向已经回收的内存地址，此情况下该指针便称野指针

野指针异常堪称`crash界`的半壁江山，相比起`NSException`而言，野指针有这么两个特点：

- `随机性强`
  
    尽管大公司已经有各种单元、行为、自动化以及人工化测试，尽量的去模拟用户的使用场景，但野指针异常总是能巧妙的避开测试，在线上大发神威。原因绝不仅仅在于测试无法覆盖所有的使用场景
    
    ![](http://upload-images.jianshu.io/upload_images/783864-9fa6e25efbe248e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    造成野指针是多样化的：首先内存被释放后不代表`内存会立刻被覆写`或者`数据受到破坏`，这时候访问这块内存也不一定会出错。其次，`多线程技术`带来了复杂的应用运行环境，在这个环境下，未加保护的数据可能是致命的。此外，`设计不够严谨的代码`同样也是造成野指针异常的重要原因之一

- `难以定位`
  
    `NSException`是高抽象层级上的封装，这意味着它可以提供更多的错误信息给我们参考。而野指针几乎出自于`C语言`层面，往往我们能获得的只有系统栈信息，单单是定位错误代码位置已经很难了，更不要说去重现修复

  ![](http://upload-images.jianshu.io/upload_images/783864-e4ae16495052073e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 定位
解决野指针最大的难点在于定位。通常线上出现了`crash`需要修复时，开发者最重要的一个步骤是重现`crash`。而上文提到了野指针的两个特性会阻碍我们定位问题，对于这两个特性，确实也能做一些对应的处理来降低它们的干扰性：

- `采集辅助信息`
  
    辅助信息包括设备信息、用户行为等信息，往往可以用来重现问题。比如用户行为可以形成`用户使用路径`，从而重现用户使用场景。而在发生`crash`时，采集当前页面信息，配合`用户使用路径`可以快速的定位到问题发生的大概位置。经过验证，`辅助信息`确实有效的减少了`系统栈`对于问题重现的干扰

- `提高野指针崩溃率`
  
    由于野指针不一定会发生崩溃这一特性，即便我们通过`堆栈信息`和`辅助信息`确定了大致范围，不代表我们能顺利的重现`crash`。一个优秀的野指针崩溃可以造成`一天开发，三天debug`，假如野指针的崩溃不是随机的，那么问题就简单的多

    ![](http://upload-images.jianshu.io/upload_images/783864-8f10d17024030d8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    `Xcode`提供了`Malloc Scribble`对已释放内存进行数据填充，从而保证野指针访问是必然崩溃的。另外，`Bugly`借鉴这一原理，通过修改`free`函数，对已释放对象进行非法数据填充，也有效的提高了野指针的崩溃率

- `Zombie Objects`
  
    `Zombie Objects`是一种完全不同的野指针调试机制，将释放的对象标记为`Zombie`对象，再次给`Zombie`对象发送消息时，发生`crash`并且输出相关的调用信息。这套机制同时定位了发生`crash`的类对象以及有相对清晰的调用栈

## 解决方案
整理一下上述的内容，可以看到目前存在`辅助信息+对象内存填充`以及`Zombie Objects`这两种主要的应对方式。拿前者来说，填充已释放对象的内存风险高，经过尝试`Xcode9`的`Malloc Scribble`启动后已经不会填充对象的内存地址。其次，填充内存需要去`hook`更加底层的`API`，这意味着对代码能力要求更高。因此，借鉴`Zombie Objects`的实现思路去定位野指针异常是一个可行的方案

### 转发
转发是一项有趣的机制，它通过在通信双方中间，插入一个中间层。发送方不再耦合接收方，它只需要将数据发送给中间层，由中间层来派发给具体的接收方。基于转发的思想，可以做许多有趣的东西：

- `消息转发`
  
    `iOS`的消息机制让我们可以给对象发送一个未注册的消息，通常这会引发`unrecognized selector`异常。但是在抛出异常之前，存在一个`消息转发`机制，允许我们重新指定消息的接收方来处理这个消息。正是这一机制实现了防`unrecognized selector crash`的可行化

    ![](http://upload-images.jianshu.io/upload_images/783864-326a3b64d8e35875.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `打破引用环`
  
    循环引用是`ARC`环境下最容易出现的内存问题，当多个对象之间的引用形成了引用环时，极有可能会导致环中的对象都无法被释放。借鉴`Proxy`的方式，可以实现破坏引用环的作用。[XXShield](https://github.com/sindrilin/XXShield)以插入`WeakProxy`层的方式实现了防`crash`

    ![](http://upload-images.jianshu.io/upload_images/783864-726601db8b3d76a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  
- `路由转发`

    组件化是项目体量达到一定程度时必须考虑的架构方案，将项目拆分基础组件和业务组件，加入中间层实现组件间解耦的效果。由于业务组件之间互不依赖，因此需要合适的方案实现组件通信，路由设计是一种常用的通信方式。各个模块实现`canOpenURL:`接口来判断是否处理对应的跳转逻辑，模块将参数信息拼接在`url`中传递：
      
    ![](http://upload-images.jianshu.io/upload_images/783864-9858d55112e0d266.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 消息发送
都说`消息发送`是`Objective-C`的核心机制，任何一个对象方法调用都会被转换成`objc_msgSend`的方式执行。这一过程中涉及到一个重要的变量：`isa`指针。多数开发者对`isa`指针停留在它指向了类的类结构本身的地址，用来表示对象的类型。但是实际上`isa`指针要比我们想想的复杂的多，比如`objc_msgSend`依赖于`isa`来完成消息的查找，通过阅读[通过汇编解读 objc_msgSend](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=5&ved=0ahUKEwjL3MvVkprXAhUFKpQKHXRrA5AQFghCMAQ&url=https%3A%2F%2Famywushu.github.io%2F2016%2F11%2F09%2F%25E9%2580%2586%25E5%2590%2591%25E7%259F%25A5%25E8%25AF%2586-%25E9%2580%259A%25E8%25BF%2587%25E6%25B1%2587%25E7%25BC%2596%25E8%25A7%25A3%25E8%25AF%25BB-objc_msgSend.html&usg=AOvVaw2-7S0WrSYzNkTZ-SwXG7hm)可以了解更详细的匹配过程：

    union isa_t {
        isa_t() { }
        isa_t(uintptr_t value) : bits(value) { }
        Class cls;
        uintptr_t bits;
        struct {
            uintptr_t indexed           : 1;
            uintptr_t has_assoc         : 1;
            uintptr_t has_cxx_dtor      : 1;
            uintptr_t shiftcls          : 33; 
            uintptr_t magic             : 6;
            uintptr_t weakly_referenced : 1;
            uintptr_t deallocating      : 1;
            uintptr_t has_sidetable_rc  : 1;
            uintptr_t extra_rc          : 19;
        };
    };
    
由于方法调用与`isa`指针相关，因此如果我们修改一个类的`isa`指针使其指向一个目标类，那么可以实现对象方法调用的拦截，也可以称作对象方法转发。我们并不能直接修改`isa`指针，但`runtime`提供了一个`object_setclass`接口允许我们动态的对某个类进行重定位

> `ClassA`被重定位成`ClassB`需要保证两个类的内存结构是对齐的，否则可能会发生超出意外的问题

一般来说我们都不应该违背重定位类的内存结构对齐原则。但在野指针问题中，对象拥有的内存被释放后是不确定状态，因此做`适当的破坏`并不一定是坏事，只是记住在最终释放对象内存时，应当再次重定位回来，防止内存泄漏的风险

### 代码实现
借鉴于`Zombie Objects`的机制，我们可以实现一套类`Zombie Proxy`机制。通过`重定位类型`的做法，在对象`dealloc`之前将其`isa`指针指向一个目标类，实现后续调用的转发。而目标类中所有的方法调用都采用`NSException`的机制抛出异常，并且输出调用对象的实际类型和调用方法帮助定位：

![](http://upload-images.jianshu.io/upload_images/783864-615cbbdccd8f5b9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

重定位后的类由于其实际用于转发的用途，更符合`Proxy`的属性，因此我将其设置为`NSProxy`的子类，多数人可能不知道`iOS`一共有`NSProxy`跟`NSObject`两个根类。另外，为了实现对`retain`等内存管理相关方法的重写，目标类应该设置为不支持`ARC`：

    @interface LXDZombieProxy : NSProxy

    @property (nonatomic, assign) Class originClass;

    @end

    @implementation LXDZombieProxy
  
    - (void)_throwMessageSentExceptionWithSelector: (SEL)selector
    {
        @throw [NSException exceptionWithName:NSInternalInconsistencyException 
                                       reason:[NSString stringWithFormat:@"(-[%@ %@]) was sent to a zombie object at address: %p", NSStringFromClass(self.originClass), NSStringFromSelector(selector), self] 
                                     userInfo:nil];
    }
    
    #define LXDZombieThrowMesssageSentException() [self _throwMessageSentExceptionWithSelector: _cmd]
    
    - (id)retain
    {
        LXDZombieThrowMesssageSentException();
        return nil;
    }
    
    - (oneway void)release
    {
        LXDZombieThrowMesssageSentException();
    }
    
    - (id)autorelease
    {
        LXDZombieThrowMesssageSentException();
        return nil;
    }
    
    - (void)dealloc
    {
        LXDZombieThrowMesssageSentException();
        [super dealloc];
    }
    
    - (NSUInteger)retainCount
    {
        LXDZombieThrowMesssageSentException();
        return 0;
    }
    
    @end

由于`iOS`的方法实际上是以`向上调用`的链式机制实现的，因此只需要`hook`掉两个根类的`dealloc`方法就能保证对对象类型的重定位。在`hook`掉`dealloc`之后有几个需要注意的点：

- `对象的释放`
  
    由于我们需要实现转发机制，这代表着本该释放的对象在类型重定位后不能被释放。随着时候时间的推移，重定位类对象的数量会越来越多。根据经验来说，一般的野指针在`30s`内被再次访问的概率很大，因此我们可以在类型重定位完成后延后`30s`释放对象。或者可以构建一个`Zombie Pool`，当内存占用达到一定大小时，使用恰当的算法淘汰

- `白名单机制`
     
    并不是所有的类对象都被监控，比如`系统私有类`、`监控相关工具类`、`明确不存在野指针的类`等。我们需要一个全局的白名单系统，来确保这些类的`dealloc`是正常执行的，无需被转发

- `潜在的crash`
     
    通过`method_setImplementation`替换`dealloc`的代码实现，由于我采用`block`转`IMP`的方式来实现的方式，会对捕获的外界对象进行引用。而对象在重定位后，任何调用都会引发`crash`，因此需要针对这种情况做对应的处理

为了满足保证对象能够在达成释放条件完成内存的回收，需要存储根类的`dealloc`原实现，以根类类名作为`key`存储在全局字典中。并且提供接口`__lxd_dealloc`来完成对象的释放工作：

    static inline void __lxd_dealloc(__unsafe_unretained id obj) {
        Class currentCls = [obj class];
        Class rootCls = currentCls;
        
        while (rootCls != [NSObject class] && rootCls != [NSProxy class]) {
            rootCls = class_getSuperclass(rootCls);
        }
        NSString *clsName = NSStringFromClass(rootCls);
        LXDDeallocPointer deallocImp = NULL;
        [[_rootClassDeallocImps objectForKey: clsName] getValue: &deallocImp];
        
        if (deallocImp != NULL) {
            deallocImp(obj);
        }
    }

    NSMutableDictionary *deallocImps = [NSMutableDictionary dictionary];
    for (Class rootClass in _rootClasses) {
        IMP originalDeallocImp = __lxd_swizzleMethodWithBlock(class_getInstanceMethod(rootClass, @selector(dealloc)), swizzledDeallocBlock);
        [deallocImps setObject: [NSValue valueWithBytes: &originalDeallocImp objCType: @encode(typeof(IMP))] forKey: NSStringFromClass(rootClass)];
    }

在对象的`dealloc`被调起之后，检测对象类型是否存在白名单中。如果存在，直接继续完成对对象的释放工作。否则的话，延后`30s`进行释放工作。为了解除`block`引用造成的`crash`，使用`NSValue`存储对象信息以及使用`__unsafe_unretained`来防止临时变量的引用：

    swizzledDeallocBlock = [^void(id obj) {
        Class currentClass = [obj class];
        NSString *clsName = NSStringFromClass(currentClass);
        /// 如果为白名单，则不重定位类的类型
        if ([__lxd_sniff_white_list() containsObject: clsName]) {
            __lxd_dealloc(obj);
        } else {
            NSValue *objVal = [NSValue valueWithBytes: &obj objCType: @encode(typeof(obj))];
            object_setClass(obj, [LXDZombieProxy class]);
            ((LXDZombieProxy *)obj).originClass = currentClass;
            
            /// 延后30秒释放对象，避免造成内存的浪费
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(30 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                __unsafe_unretained id deallocObj = nil;
                [objVal getValue: &deallocObj];
                object_setClass(deallocObj, currentClass);
                __lxd_dealloc(deallocObj);
            });
        }
    } copy];

具体的实现代码可以下载[LXDZombieSniffer](https://github.com/sindrilin/LXDZombieSniffer)

## 疑难问题
野指针问题是访问了非法内存导致的`crash`，也就是说要符合两个条件：`内存非法`以及`指针地址不为NULL`。在`iOS`中存在三种不同修饰的指针：

- `__strong`
     
    默认修饰符。修饰的指针在赋值之后，会对指向的对象执行一次`retain`操作，指针不因对象的生命周期变化而改变

- `__unsafed_unretained`
  
    非安全对象指针修饰符。修饰的指针不会持有指向对象，也不因对象的生命周期发生变化而改变，等同于`assign`

- `__weak`
  
    弱对象指针修饰符。修饰的指针不会持有指向对象，在对象的生命周期结束并且内存被回收时，修饰的指针内容会被重置为`nil`
  
根据野指针异常的引发条件来说，三种修饰指针只有`__strong`和`__unsafed_unretained`可以导致野指针访问异常。但是在使用`类别重定位`之后，本该释放的对象会被延时或者不释放，也就是本该被重置的弱指针也不会发生重置，这时使用弱指针访问对象应该会被转发到`ZombieProxy`当中发生`crash`：

    __weak id weakObj = nil;
    @autoreleasepool {
        NSObject *obj = [NSObject new];
        weakObj = obj;
    }
    /// The operate should be crashed
    NSLog(@"%@", weakObj);
    
然而在上面的测试中，发现即便对象被重定位为`Zombie`并且被阻止释放之后，`weakObj`依旧被成功的设置成了`nil`。然后经过[objc_runtime](https://github.com/RetVal/objc-runtime)源码运行和添加断点测试之后，也没有`weak`指针被重置的调用。甚至使用了`LLVM`的`watch set var weakObj`监控弱指针，依旧无法找到调用。但`weakObj`在`dealloc`调用之后，不管对象有没有被释放，都被重置成了`nil`。这也是截止文章出来为止，匪夷所思的疑难杂症

## 参考
[如何定位Obj-C野指针随机Crash(一)](https://dev.qq.com/topic/59141e56ca95d00d727ba750)

[如何定位Obj-C野指针随机Crash(二)](https://dev.qq.com/topic/59142d61ca95d00d727ba752)

[如何定位Obj-C野指针随机Crash(三)](https://dev.qq.com/topic/5915134b75d11c055ca7fca0)

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)

