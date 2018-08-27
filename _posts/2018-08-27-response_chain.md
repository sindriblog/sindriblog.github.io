---
title: 分析实现-谈谈响应链
date: 2018-08-27 08:00
tags: 分析实现
---

当用户的手指在屏幕上的某一点按下时，屏幕接收到点击信号将点击位置转换成具体坐标，然后本次点击被包装成一个点击事件`UIEvent`。最终会存在某个视图响应本次事件进行处理，而为`UIEvent`查找响应视图的过程被称为`响应链查找`，在整个过程中有两个至关重要的类：`UIResponder`和`UIView`

## 响应者
响应者是可以处理事件的具体对象，一个响应者应当是`UIResponder`或其子类的实例对象。从设计上来看，`UIResponder`主要提供了三类接口：

- 向上查询响应者的接口，体现在`nextResponder`这个唯一的接口
- 用户操作的处理接口，包括`touch`、`press`和`remote`三类事件的处理
- 是否具备处理`action`的能力，以及为其找到`target`的能力

总体来说`UIResponder`为整个事件查找过程提供了处理能力

### 视图
视图是展示在界面上的可视元素，包括不限于`文本`、`按钮`、`图片`等可见样式。虽然`UIResponder`提供了让对象响应事件的处理能力，但拥有处理事件能力的`responder`是无法被用户观察到的，换句话说，用户也无法点击这些有处理能力的对象，因此`UIView`提供了一个可视化的载体，从接口上来看`UIView`提供了三方面的能力：

- 视图树结构。虽然`responder`也存在相同的树状结构，但其必须依托于可视载体进行表达
- 可视化内容。通过`frame`等属性决定视图在屏幕上的可视范围，提供了点击坐标和响应可视对象的关联能力
- 内容布局重绘。视图渲染到屏幕上虽然很复杂，但按照不同的`layout`方式提供了不同阶段的重绘调起接口，使得子类具有很强的定制性

视图树的结构如下，由于`UIView`是`UIResponder`的子类，可以通过`nextResponder`访问到父级视图，但由于`responder`并不全是具备可视载体的对象，通过`nextResponder`向上查找的方式可能会导致无法通过位置计算的方式查找响应者
![](https://upload-images.jianshu.io/upload_images/783864-0301504ec3344805.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 查找响应者
讲了这么多，也该聊聊查找响应者的过程了。前面说了，`responder`决定了对象具有响应处理的能力，而`UIView`才是提供了可视载体和点击坐标关联的能力。换句话说，查找响应者实际上是查找点击坐标落点位置在其可视范围内且其具备处理事件能力的对象，按照官方话来说就是既要`responde`又要`view`的对象。因为需要先找到响应者，才能有进一步的处理，所以直接从后者的接口找起，两个`api`：

    /// 检测坐标点是否落在当前视图范围内
    - (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
    /// 查找响应处理事件的最终视图
    - (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;

通过`exchange`掉第一个方法的实现，很轻易的就能得出响应者查找的顺序：

    - (BOOL)sl_pointInside: (CGPoint)point withEvent: (UIEvent *)event {
        BOOL res = [self sl_pointInside: point withEvent: event];
        if (res) {
            NSLog(@"[%@ can answer]", self.class);
        } else {
            NSLog(@"non answer in %@", self.class);
        }
        return res;
    }

![](https://upload-images.jianshu.io/upload_images/783864-c9039220607d556e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如图创建相同的布局结构，然后点击`BView`，得到的日志：

    [UIStatusBarWindow can answer]
    non answer in UIStatusBar
    non answer in UIStatusBarForegroundView
    non answer in UIStatusBarServiceItemView
    non answer in UIStatusBarDataNetworkItemView
    non answer in UIStatusBarBatteryItemView
    non answer in UIStatusBarTimeItemView
    [UIWindow can answer]
    [UIView can answer]
    non answer in CView
    [AView can answer]
    [BView can answer]

通过日志输出可以看出查找顺序优先级有两条：

1. 优先级更高的`window`优先匹配
2. 同一个`window`中从父视图向子视图寻找

通过`pointInside:`确定了点击坐标落在哪些视图的范围后，会继续调用另一个方法来找寻真正的响应者。同样`hook`掉这个方法，从日志来看父级视图会调用子视图，最终以递归的格式输出：

    - (UIView *)sl_hitTest: (CGPoint)point withEvent: (UIEvent *)event {
        UIView *res = [self sl_hitTest: point withEvent: event];
        NSLog(@"hit view is: %@ and self is: %@", res.class, self.class);
        return res;
    }

    /// 输出日志
    hit view is: (null) and self is: CView
    hit view is: BView and self is: BView
    hit view is: BView and self is: AView
    hit view is: BView and self is: UIView
    hit view is: BView and self is: UIWindow

当确定了是`CView`是最后一个落点视图时，会以`CView`为起始点，向上寻找事件的响应者，这个查找过程由一个个`responder`链接起来，这也是`响应链`名字的来由。另外，基于树结构的视图层级，只需要我们持有根节点，就能遍历整棵树，这也是为什么搜索过程是从`window`开始的

### 响应者处理
经过`pointInside:`和`hitTest:`两个方法会确定点击位置最上方的可视响应者，但并不代表了这个`responder`会处理本次事件。基于上面的界面`demo`，在`AView`中实现`touches`方法：

    @implementation AView

    - (void)touchesBegan: (NSSet<UITouch *> *)touches withEvent: (UIEvent *)event {
        NSLog(@"A began");
    }

    - (void)touchesCancelled: (NSSet<UITouch *> *)touches withEvent: (UIEvent *)event {
        NSLog(@"A canceled");
    }
    
    - (void)touchesEnded: (NSSet<UITouch *> *)touches withEvent: (UIEvent *)event {
        NSLog(@"A ended");
    }
    
    @end

![](https://upload-images.jianshu.io/upload_images/783864-6c222f90259cbc03.gif?imageMogr2/auto-orient/strip)

前面说过`UIResponder`提供了用户操作的处理接口，但很明显`touches`系列的接口默认是`未实现的`，因此`BView`即便成为了响应链上的最上层节点，依旧无法处理点击事件，而是沿着响应链查找响应者：

    void handleTouch(UIResponder *responder, NSSet<UITouch *> *touches UIEvent *event) {
        if (!responder) {
            return;
        }
        if ([responder respondsToSelector: @selector(touchesBegan:withEvent:)]) {
            [responder touchesBegan: touches withEvent: event];
        } else {
            handleTouch(responder.nextResponder, touches, event);
        }
    }

另外一个有趣的地方是手势拦截，除了实现`touches`系列方法让`view`提供响应能力之外，我们还可以主动的在视图上添加手势进行回调：

    - (void)viewDidLoad {
        [super viewDidLoad];
        [_a addGestureRecognizer: [[UITapGestureRecognizer alloc] initWithTarget: self action: @selector(clickedA:)]];
    }

    - (void)clickedA: (UITapGestureRecognizer *)tap {
        NSLog(@"gesture clicked A");
    }

    /// 日志输出
    A began
    gesture clicked A
    A canceled

从日志可以看到手势的处理是在`touchesBegan`之后，并且执行之后会中断原有的`touches`调用链，因此可以确定即便是手势动作，最终依旧由`UIResponder`进行事件分发处理

### 小结
触屏事件的处理被分成两个阶段：`查找响应者`和`响应者处理`，分别由`UIView`和 `UIResponder`提供了功能上的支持。另外由于`pointInside:`和`hitTest:`这两个关键接口是对外暴露的，因此通过`hook`或者`inherit`的方式来修改这两个方法，可以使得视图的可响应范围大于显示范围

## 应用
最近有个需求需要在`tabbar`的位置上方弹出气泡，且允许用户点击气泡发生交互事件。从视图层来分析，`tabbar`被嵌套在多层尺寸等同于菜单栏的`view`当中：

![](https://upload-images.jianshu.io/upload_images/783864-f7557f75b6029618.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果要实现从`item`弹起气泡并且可交互，有两个可行的方案：

1. 将气泡添加到`ViewController`的视图上
2. 修改`tabbar`的响应链查找接口，实现显示范围外的可点击处理

考虑到如果项目中存在相同的弹出需求可能会导致写了一堆重复代码，所以将`弹出动作`和`触屏判断`给封装起来，通过`hook`的方式来实现弹出气泡自动化触屏检测功能

### 接口设计
依照最少接口原则，只暴露两个弹出接口：

    /*!
     *  @enum   SLViewDirection
     *  弹窗视图所在的方向（dismiss和pop应当保持一致）
     */
    typedef NS_ENUM(NSInteger, SLViewDirection)
    {
        SLViewDirectionCenter,
        SLViewDirectionTop,
        SLViewDirectionLeft,
        SLViewDirectionBottom,
        SLViewDirectionRight
    };
    
    /*!
     *  @category   UIView+SLFreedomPop
     *  自由弹窗扩展
     */
    @interface UIView (SLFreedomPop)
    
    /*!
     *  @method sl_popView:
     *  居中弹出view
     *  @param  view    要弹出的view
     */
    - (void)sl_popView: (UIView *)view;
    
    /*!
     *  @method sl_popView:WithDirection:
     *  控制弹窗方向
     *  @param  view        要弹出的view
     *  @param  direction   弹出方向
     */
    - (void)sl_popView: (UIView *)view withDirection: (SLViewDirection)direction;
    
    @end

### 落点检测
考虑到两个问题：

1. 视图可能存在多个超出显示范围的子视图
2. 视图存在超出父级视图显示范围的子视图

针对这两个问题，解决方案分别是：

1. 用视图作`key`，存储一个`extraRect`的列表
2. 当弹出视图超出自身范围时，向父级视图调用，确保父级视图能处理响应

最终检测代码如下：

    #define SLRectOverflow(subrect, rect) \
        subrect.origin.x < 0 || \
        subrect.origin.y < 0 || \
        CGRectGetMaxX(subrect) > CGRectGetWidth(rect) ||   \
        CGRectGetMaxY(subrect) > CGRectGetHeight(rect)

    #pragma mark - Private
    - (BOOL)_sl_pointInsideExtraRects: (CGPoint)point {
        NSArray *extraRects = [self extraHitRects].allValues;
        if (extraRects.count == 0) {
            return NO;
        }
        
        for (NSSet *rects in extraRects) {
            for (NSString *rectStr in rects) {
                if (CGRectContainsPoint(CGRectFromString(rectStr), point)) {
                    return YES;
                }
            }
        }
        return NO;
    }

    #pragma mark - Rects
    - (void)_sl_addExtraRect: (CGRect)extraRect inSubview: (UIView *)subview {
        CGRect curRect = [subview convertRect: extraRect toView: self];
        if (SLRectOverflow(curRect, self.frame)) {
            [self _sl_expandExtraRects: curRect forKey: [NSValue valueWithBytes: &subview objCType: @encode(typeof(subview))]];
            [self.superview _sl_addExtraRect: curRect inSubview: self];
        }
    }
    
    #pragma mark - Hook
    - (BOOL)sl_pointInside: (CGPoint)point withEvent: (UIEvent *)event {
        BOOL res = [self sl_pointInside: point withEvent: event];
        if (!res) {
            return [self _sl_pointInsideExtraRects: point];
        }
        return res;
    }
    
    - (UIView *)sl_hitTest: (CGPoint)point withEvent: (UIEvent *)event {
        UIView *res = [self sl_hitTest: point withEvent: event];
        if (!res) {
            if ([self _sl_pointInsideExtraRects: point]) {
                return self;
            }
        }
        return res;
    }

### 运行效果
红色视图弹出绿色视图，超出自身和父视图的显示范围，点击有效：
![](https://upload-images.jianshu.io/upload_images/783864-211f2a575f6cdc0a.gif?imageMogr2/auto-orient/strip)

### 定制改进
由于核心功能在于`event handle`，目前代码只提供了简单的弹出接口，需要进一步扩充弹出接口能力的可以考虑两点改进：

- 增加`configuration`来定制弹窗样式
- 提供`animation`来完成自定义的动画

文章的[源码地址](https://github.com/sindrilin/SLFreedomPop)

![关注我的公众号获取更新信息](https://upload-images.jianshu.io/upload_images/783864-5f15782c42a970c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

