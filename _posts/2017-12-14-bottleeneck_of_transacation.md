---
title: 质量监控-隐式动画的性能瓶颈
date: 2017-12-14 08:00:00
tags:
- 质量监控
---

`隐式动画`实现的背后体现了核心动画精心设计的许多机制。在`layer`的属性发生改变之后，会向它的代理方请求一个`CAAction`行为来完成后续的工作，系统允许代理方返回`nil`指针。一旦这么做，修改属性的工作最终移交给`CATransaction`处理，由修改的属性值决定是否自动生成一个`CABasicAnimation`。如果满足，此时隐式动画将被触发。

### 关于CATransaction
在核心动画中，每个图层的修改都是事务`CATransaction`的一部分，它可以同时对多个`layer`的属性进行修改，然后成批的将将多个图层树包装起来，一次性发送到渲染服务进程。`CATransaction`事务对象被分为`implicit`和`explicit`两种类型，分别对应`隐式`和`显式`。`implicit transaction`会被投递到线程的下一个`runloop`完成处理：

> Core Animation supports two types of transactions: implicit transactions and explicit transactions. Implicit transactions are created automatically when the layer tree is modified by a thread without an active transaction and are committed automatically when the thread's runloop next iterates.

默认情况下，`CATransaction`会在背后独立完成图层树属性计算的工作。系统提供`API`来显式的使用事务类，并且手动提交给渲染服务进程，这种做法被称作`推进过渡`。`推进过渡`会生成一个默认时长为`0.25s`时长的动画效果来完成属性值的修改。下面代码会在`0.25s`内将图层放大一倍：

    [CATransaction begin];
    self.circle.transform = CATransform3DScale(CATransform3DIdentity, 2, 2, 1);
    [CATransaction commit];

### layer如何实现动画
图层属性被修改时，会朝着自己的代理对象请求一个`CAAction`行为来帮助自己完成属性修改的行为。代理方法`actionForLayer:forKey:`允许三种返回的数据格式来完成不同的修改动作：

- 空对象
    
    `UIView`在响应代理时默认会返回一个`NSNull`对象，表示属性修改后，不实现任何的动作，根据修改后的属性值直接更新视图。但`UIView`不总是会返回空对象，如果`layer`的修改发生在`[UIView animatedXXX]`接口的`block`中，每一个修改的属性值`UIView`都会返回对应的`CABasicAnimation`对象来进行动画修改

- `nil`
    
    手动创建并添加到视图上的`CALayer`或其子类在属性修改时，没有获取到具体的修改行为。此时被修改的属性会被`CATransaction`记录，最终在下一个`runloop`的回调中生成动画来响应本次属性修改。由于这个过程非开发者主动完成的，因此这种动画被称作`隐式动画`

- `CAAction`的子类

    如果返回的是`CAAction`对象，会直接开始动画来响应图层属性的修改。一般返回的对象多为`CABasicAnimation`类型，对象中包装了`动画时长`、`动画初始/结束状态`、`动画时间曲线`等关键信息。当`CAAction`对象被返回时，会立刻执行动作来响应本次属性修改

### 了解隐式动画的必要
首先，`隐式动画`是相对于`显式动画`而言的，属于被动实现。由于`显式动画`是主动实现的，因此在实现这些动画的时候，我们会去考虑动画是否流畅，动画前后是否会有卡帧，也会不断的运行来保证动画效果如预期完成。而`隐式动画`多属于系统自己完成的动画效果，提供给我们的可调试空间也很小，这也导致了开发者对它的重视不够，从而阻碍了进一步深入学习的可能性。

其次，和用户直接进行交互的就是`UI`元素。在发生卡帧、掉帧的性能问题时，用户对静止界面和动画的感知是完全不同的。即便只有`1`帧页面丢失，在动画中也能轻易的被用户捕捉。举个例子，当用户按下按钮，应用推迟了`1、2`帧才开始跳转。又或者是在界面跳转时丢失帧数据，具体表现为卡帧，此时用户对于卡顿的感知是远远大于平常的，因此了解`隐式动画`过程中如何发生卡顿是很有必要的。

### 隐式动画何时开始
隐式动画的修改最终由`CATransaction`事务完成，它在主线程的`runloop`注册了一个监听者，具体回调发生在`before waiting`阶段。在回调中会将所有`implicit transactions`以动画的形式展示。虽然苹果文档没有明说具体的回调时机，但通过简单的测试可以定位`transaction`的回调时间：通过注册两个`runloop`监听者，回调优先级分别设为`NSIntegerMax`和`NSIntegerMin`，监控最早和最晚的回调阶段，并且在对应位置添加断点，查看断点前后图层是否更新：

    - (void)viewDidLoad {
        [super viewDidLoad];
        
        CFRunLoopObserverContext ctx = { 0, (__bridge void *)self, NULL, NULL };
        CFRunLoopObserverRef allActivitiesObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopBeforeWaiting, YES, NSIntegerMin, &__runloop_callback, &ctx);
        CFRunLoopAddObserver(CFRunLoopGetCurrent(), allActivitiesObserver, kCFRunLoopCommonModes);
        
        CFRunLoopObserverRef beforeWaitingObserver = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopBeforeWaiting, YES, NSIntegerMax, &__runloop_before_waiting_callback, &ctx);
        CFRunLoopAddObserver(CFRunLoopGetCurrent(), beforeWaitingObserver, kCFRunLoopCommonModes);
    }
    
手动创建的`CALayer`在属性修改会产生隐式动画，将`layer`增加到视图层级上后，点击按钮来修改它的`transform`属性，并且观察断点前后的效果：

    self.circle = [CAShapeLayer layer];
    self.circle.delegate = self;
    self.circle.anchorPoint = CGPointMake(0.5, 0.5);
    self.circle.fillColor = [UIColor orangeColor].CGColor;
    self.circle.path = [UIBezierPath bezierPathWithOvalInRect: CGRectMake(CGRectGetMidX([UIScreen mainScreen].bounds) - 50, 80, 100, 100)].CGPath;
    [self.view.layer addSublayer: self.circle];

通过断点和界面显示可以看到在`Before Waiting`阶段的两次回调之间，`transaction`完成了属性修改的渲染任务（在`DEBUG+断点`状态下，隐式动画不能很好的完成动画效果）：

![](https://user-gold-cdn.xitu.io/2017/12/15/16059375e2064267?w=1240&h=773&f=jpeg&s=82512)

![](https://user-gold-cdn.xitu.io/2017/12/15/160593761443aa0d?w=1240&h=575&f=jpeg&s=74566)

通过上面的测试可以确定`transaction`的事务处理确实发生在`before waiting`阶段。但由于注册`observer`时传入的优先级可以影响回调顺序，为了排除回调顺序可能对测试的干扰，可以通过`hook`掉`CFRunLoopAddObserver`这一注册函数，来获取已有的所有注册`before waiting`的回调信息：

    void new_runloopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode) {
        CFOptionFlags activities = CFRunLoopObserverGetActivities(observer);
        if (activities & kCFRunLoopBeforeWaiting) {
            CFRunLoopObserverContext context;
            CFRunLoopObserverGetContext(observer, &context);
            void *info = context.info;
            NSLog(@"%d, %@", CFRunLoopObserverGetOrder(observer), (__bridge id)info);
        }
        origin_CFRunLoopAddObserver(rl, observer, mode);
    }
    
运行后应用注册了`5`个包含`before waiting`状态的`observer`，优先级分别是最小为`0`，最大为`2147483647`，也就是`0 ~ 2^31-1`，处于`NSIntegerMin`和`NSIntergerMax`之间，足以确定测试的正确性。

### 隐式动画的性能瓶颈
通过上面的测试，可知`layer`的隐式动画发生在`before waiting`这一阶段。那么理论上来说，假如在两个监听回调之间发生了卡顿，应该会对动画效果造成影响。另外，卡顿的时机可能也会影响动画的效果。分别在`transaction`的回调让主线程进入休眠来测试不同时机的卡顿对动画造成的效果，上面的测试证明了注册的两个已有回调可以用于制作不同时机的卡顿：

    NSLog(@"ready sleep");
    [NSThread sleepForTimeInterval: 1];
    NSLog(@"after sleep");

先于`CATransaction`回调发生卡顿。点击按钮后，界面卡顿`1s`，然后才开始执行动画。期间多次点击按钮无效：
    
![](https://user-gold-cdn.xitu.io/2017/12/15/16059375d8247d30?w=797&h=555&f=gif&s=216874)

后于`CATransaction`回调发生卡顿。点击按钮后，动画立刻开始执行。界面会停止响应`1s`，同样卡顿期间不响应点击。动画存在卡帧现象，但不严重：

![](https://user-gold-cdn.xitu.io/2017/12/15/16059375d83f2554?w=797&h=555&f=gif&s=189996)

在`transaction`前后制作卡顿确实产生了不同的效果，但是即便更换卡顿的时机，动画效果仍是比较流畅的，这证明了渲染、展示过程和主线程可能是并发执行的。实际上在`WWDC2014`的视频中有对图层渲染过程的详细讲述，`隐式动画`遵循这样的渲染过程。图层渲染过程分为三个阶段：

![](https://user-gold-cdn.xitu.io/2017/12/15/16059375dc368d95?w=692&h=343&f=jpeg&s=18871)

1. `Commit Transaction + Decode`
  
    `transaction`此时会将被修改的一个或者多个`layer`的属性统一计算，更新`modelLayer`属性，然后将图层信息整合提交渲染服务进程。渲染服务进程反序列化获取渲染树信息，并准备开始渲染
    
2. `Draw Calls + Render`

    渲染服务进程根据渲染树信息，计算出动画的帧数和图层信息。此时`GPU`利用渲染树开始合成位图并准备展示到屏幕上

3. `Display`

    将渲染好的位图信息展示到屏幕上，如果存在动画则逐帧展示。如果在`transaction`后发生卡顿，会对动画展示造成一定的影响，但影响程度相对较低

结合渲染服务进程的工作流程，可以知道实际上`transaction`的工作是`1`，在`transaction`回调结束时已经将图层树提交给渲染服务进程了，因此之后即便主线程发生卡顿，也不会影响渲染服务进程的工作。而早于`transaction`回调发生的卡顿会导致应用不能将图层树及时的提交到渲染服务进程，从而造成了动画开始前的界面停滞现象。

### 显式动画何时开始
说完了`隐式动画`如何开始、瓶颈等信息，对应的也理当说说`显式动画`。虽然直接响应属性修改是`显式动画`的最大特点，但通过已有的测试可以直接证明这一点。修改`CALayerDelegate`的代理方法，主动返回一个`CABasicAnimation`对象：

    #pragma mark - CALayerDelegate
    - (id<CAAction>)actionForLayer: (CALayer *)layer forKey: (NSString *)event {
        CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath: event];
        CGFloat randomScale = (arc4random() % 20 + 1) * 0.1;
        animation.toValue = [NSValue valueWithCATransform3D: CATransform3DScale(CATransform3DIdentity, randomScale, randomScale, 1)];
        return animation;
    }
    
同样添加断点进行测试，运行后可以看到在动画开始之后，断点才会停下来。也可以确定虽然`transaction`虽然也负责了`显式动画`的渲染事务，但会立即`commit`到渲染服务进程响应属性修改。

### 转场卡顿
默认的转场动画实际上也是由`transaction`来完成的，属于隐式动画。通过`hook`掉获取`CAAction`的代理方法，在忽略掉`nil`和`NSNull`的无效返回值后，一个`push`跳转动画总共涉及到了三个`CAAction`子类。从类名上来看`_UIViewAdditiveAnimationAction`是和转场动画关联最密切的子类，也证明了系统默认的转场跳转实际上也是交给了`transaction`机制来处理的。另外从`log`上的执行来看，转场实际上也属于`隐式动画`：

![](https://user-gold-cdn.xitu.io/2017/12/15/16059375dfa8e7a0?w=1064&h=420&f=jpeg&s=91737)

转场卡顿从效果上看能分为`转场前卡顿`和`转场后卡顿`，后者属于常见的的转场性能瓶颈，大多由于新界面视图层级复杂、大量`IO`等工作导致，是最容易定位的一类问题。而前者属于少见，且不容易定位的卡顿现象之一。结合上面的测试，如果发生了`转场前卡顿`，那么说明渲染工作在`1`开始之前就发生了卡顿。

![](https://user-gold-cdn.xitu.io/2017/12/15/16059375ddec80e9?w=269&h=555&f=gif&s=33643)

在上面的`log`中可以看到`viewDidLoad`和`viewWillAppear`的调用同样处在`before waiting`阶段。假设这两个方法的调用时机在`transaction`前面，那么一旦两个方法发生了卡顿，肯定会跳转动画卡帧后执行的效果。通过分别在两个方法中添加`sleep`操作测试，还原了`gif`的卡顿效果。因此可以得出转场动画过程中的流程：

    view did load --> view will appear --> CATransaction callback --> animate
    
### 补充
虽然苹果文档和测试结果都说明了一件事情：`transaction`的回调处在`before waiting`阶段，但是否存在可能：`runloop`无法进入`before waiting`呢？实际上这种可能是完全存在的，根据苹果文档中的描述，下图可以用来表示`runloop`的内部逻辑：

![](https://user-gold-cdn.xitu.io/2017/12/14/160558dc91def8b8?w=1294&h=996&f=png&s=229757)

假如`runloop`中一直有`source1`事件，那么会一直在`2、3、4、5、9`之间循环处理。而`touches`发生时，就是典型的持续`source1`事件环境。换句话说，如果用户一直在滚动列表，那么`before waiting`将不会到来。但实际在应用使用中，即便是手指不离开屏幕，`cell`依旧能够展示各种动画。因此可以推断出`transaction`至少还注册了`UITracking`这个模式下的`runloop`监听处理，感兴趣的同学可以在滚动列表上采用类似的手段测试具体的处理时机。

### 结论
由于隐式动画的特殊性质，我们与之打交道的地方基本在`页面跳转`环节，一旦这个过程发生了卡顿，无论是`跳转前卡顿`或者是`跳转后卡顿`，都会使得应用的体验大打折扣。总结了一下，在日常开发中，我们与隐式动画打交道时记住几点：

- 隐式动画开始前的卡顿是因为`CATransaction`回调前其他任务占用了大量的`CPU`资源，通过懒加载、延后加载、异步执行可以有效的避免这个问题

- `viewDidLoad`和`viewWillAppear`是一丘之貉，它们都会导致转场动画前的卡顿。所以如果你将前者的工作放到后者执行，并没有任何作用

- 动画在开始之后，即便是应用发生卡顿，对动画的影响也要低于先于`transaction`的卡顿。因此如果你不知道如何优化动画前的烂摊子，那么放到动画开始之后吧


![关注我的公众号获取更新信息](https://user-gold-cdn.xitu.io/2018/8/21/1655b3a6f7d188a8?w=430&h=430&f=jpeg&s=23750)

