---
title: 质量监控-卡顿检测
tags:
- 质量监控
---

不管是应用秒变幻灯片，还是启动过久被杀，基本都是开发者必经的体验。就像没人希望堵车一样，卡顿永远是不受用户欢迎的，所以如何发现卡顿是开发者需要直面的难题。虽然导致卡顿的原因有很多，但卡顿的表现总是大同小异。如果把卡顿当做病症看待，两者分别对应所谓的本与标。要检测卡顿，无论是标或本都可以下手，但都需要深入的学习

## instruments与性能
在开发阶段，使用内置的性能工具`instruments`来检测性能问题是最佳的选择。与应用运行性能关联最紧密的两个硬件`CPU`和`GPU`，前者用于执行程序指令，针对代码的处理逻辑；后者用于大量计算，针对图像信息的渲染。正常情况下，`CPU`会周期性的提交要渲染的图像信息给`GPU`处理，保证视图的更新。一旦其中之一响应不过来，就会表现为卡顿。因此多数情况下用到的工具是检测`GPU`负载的`Core Animation`，以及检测`CPU`处理效率的`Time Profiler`

![](https://upload-images.jianshu.io/upload_images/783864-ae90fc9fe3038bd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于`CPU`提交图像信息是在主线程执行的，会影响到`CPU`性能的诱因包括以下：

1. 发生在主线程的`I/O`任务
2. 过多的线程抢占`CPU`资源
3. 温度过高导致的`CPU`降频

而影响`GPU`的因素较为客观，难以针对做代码上的优化，包括：

1. 显存频率
2. 渲染算法
3. 大计算量

本文旨在介绍如何去检测卡顿，而非如何解决卡顿，因此如果对上面列出的诱因有兴趣的读者可以自行阅读相关文章书籍

## 卡顿检测
检测的方案根据线程是否相关分为两大类：

- 执行耗时任务会导致`CPU`短时间无法响应其他任务，检测任务耗时来判断是否可能导致卡顿
- 由于卡顿直接表现为操作无响应，界面动画迟缓，检测主线程是否能响应任务来判断是否卡顿

与主线程相关的检测方案包括：

1. `fps`
2. `ping`
3. `runloop`

与主线程不相关的检测包括：

1. `stack backtrace`
2. `msgSend observe`

### 衡量指标
不同方案的检测原理和实现机制都不同，为了更好的选择所需的方案，需要建立一套衡量指标来对方案进行对比，个人总结的衡量指标包括四项：

- 卡顿反馈

    卡顿发生时，检测方案是否能及时、直观的反馈出本次卡顿

- 采集精度

    卡顿发生时，检测方案能否采集到充足的信息来做定位追溯

- 性能损耗

    维持检测所需的`CPU`占用、内存使用是否会引入额外的问题

- 实现成本

    检测方案是否易于实现，代码的维护成本与稳定性等

### fps
通常情况下，屏幕会保持`60hz/s`的刷新速度，每次刷新时会发出一个屏幕刷新信号，`CADisplayLink`允许我们注册一个与刷新信号同步的回调处理。可以通过屏幕刷新机制来展示`fps`值：

    - (void)startFpsMonitoring {
        WeakProxy *proxy = [WeakProxy proxyWithClient: self];
        self.fpsDisplay = [CADisplayLink displayLinkWithTarget: proxy selector: @selector(displayFps:)];
        [self.fpsDisplay addToRunLoop: [NSRunLoop mainRunLoop] forMode: NSRunLoopCommonModes];
    }
    
    - (void)displayFps: (CADisplayLink *)fpsDisplay {
        _count++;
        CFAbsoluteTime threshold = CFAbsoluteTimeGetCurrent() - _lastUpadateTime;
        if (threshold >= 1.0) {
            [FPSDisplayer updateFps: (_count / threshold)];
            _lastUpadateTime = CFAbsoluteTimeGetCurrent();
        }
    }
    
| 指标 |  |
| --- | --- |
| 卡顿反馈 | 卡顿发生时，`fps`会有明显下滑。但转场动画等特殊场景也存在下滑情况。高 |
| 采集精度 | 回调总是需要`cpu`空闲才能处理，无法及时采集调用栈信息。低 |
| 性能损耗 | 监听屏幕刷新会频繁唤醒`runloop`，闲置状态下有一定的损耗。中低 |
| 实现成本 | 单纯的采用`CADisplayLink`实现。低 |
| 结论 | 更适用于开发阶段，线上可作为辅助手段 |

### ping
`ping`是一种常用的网络测试工具，用来测试数据包是否能到达`ip`地址。在卡顿发生的时候，主线程会出现`短时间内无响应`这一表现，基于`ping`的思路从子线程尝试通信主线程来获取主线程的卡顿延时：

    @interface PingThread : NSThread
    ......
    @end

    @implementation PingThread
    
    - (void)main {
        [self pingMainThread];
    }
    
    - (void)pingMainThread {
        while (!self.cancelled) {
            @autoreleasepool {
                dispatch_async(dispatch_get_main_queue(), ^{
                    [_lock unlock];
                });
                
                CFAbsoluteTime pingTime = CFAbsoluteTimeGetCurrent();
                NSArray *callSymbols = [StackBacktrace backtraceMainThread];
                [_lock lock];
                if (CFAbsoluteTimeGetCurrent() - pingTime >= _threshold) {
                    ......
                }
                [NSThread sleepForTimeInterval: _interval];
            }
        }
    }
    
    @end

| 指标 |  |
| --- | --- |
| 卡顿反馈 | 主线程出现堵塞直到空闲期间都无法回包，但在`ping`之间的卡顿存在漏查情况。中高 |
| 采集精度 | 子线程在`ping`前能获取主线程准确的调用栈信息。中高 |
| 性能损耗 | 需要常驻线程和采集调用栈。中 |
| 实现成本 | 需要维护一个常驻线程，以及对象的内存控制。中低 |
| 结论 | 监控能力、性能损耗和`ping`频率都成正比，监控效果强 |

### runloop
作为和主线程相关的最后一个方案，基于`runloop`的检测和`fps`的方案非常相似，都需要依赖于主线程的`runloop`。由于`runloop`会调起同步屏幕刷新的`callback`，如果`loop`的间隔大于`16.67ms`，`fps`自然达不到`60hz`。而在一个`loop`当中存在多个阶段，可以监控每一个阶段停留了多长时间：

    - (void)startRunLoopMonitoring {
        CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
            if (CFAbsoluteTimeGetCurrent() - _lastActivityTime >= _threshold) {
                ......
                _lastActivityTime = CFAbsoluteTimeGetCurrent();
            }
        });
        CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    }

| 指标 |  |
| --- | --- |
| 卡顿反馈 | `runloop`的不同阶段把时间分片，如果某个时间片太长，基本认定发生了卡顿。此外应用闲置状态常驻`beforeWaiting`阶段，此阶段存在误报可能。中 |
| 采集精度 | 和`fps`类似的，依附于主线程`callback`的方案缺少准确采集调用栈的时机，但优于`fps`检测方案。中低 |
| 性能损耗 | 此方案不会频繁唤醒`runloop`，相较于`fps`性能更佳。低 |
| 实现成本 | 需要注册`runloop observer`。中低 |
| 结论 | 综合性能优于`fps`，但反馈表现不足，只适合作为辅助工具使用 |

### stack backtrace
代码质量不够好的方法可能会在一段时间内持续占用`CPU`的资源，换句话说在一段时间内，调用栈总是停留在执行某个地址指令的状态。由于函数调用会发生入栈行为，如果比对两次调用栈的符号信息，前者是后者的符号子集时，可以认为出现了`卡顿恶鬼`：

    @interface StackBacktrace : NSThread
    ......
    @end

    @implementation StackBacktrace
    
    - (void)main {
        [self backtraceStack];
    }
    
    - (void)backtraceStack {
        while (!self.cancelled) {
            @autoreleasepool {
                NSSet *curSymbols = [NSSet setWithArray: [StackBacktrace backtraceMainThread]];
                if ([_saveSymbols isSubsetOfSet: curSymbols]) {
                    ......
                }
                _saveSymbols = curSymbols;
                [NSThread sleepForTimeInterval: _interval];
            }
        }
    }
    
    @end

| 指标 |  |
| --- | --- |
| 卡顿反馈 | 由于符号地址的唯一性，调用栈比对的准确性高。但需要排除闲置状态下的调用栈信息。高 |
| 采集精度 | 直接通过调用栈符号信息比对可以准确的获取调用栈信息。高 |
| 性能损耗 | 需要频繁获取调用栈，需要考虑延后符号化的时机减少损耗。中高 |
| 实现成本 | 需要维护常驻线程和调用栈追溯算法。中高 |
| 结论 | 准确率很高的工具，适用面广 |

### msgSend observe
`OC`方法的调用最终转换成`msgSend`的调用执行，通过在函数前后插入自定义的函数调用，维护一个函数栈结构可以获取每一个`OC`方法的调用耗时，以此进行性能分析与优化：

    #define save() \
    __asm volatile ( \
        "stp x8, x9, [sp, #-16]!\n" \
        "stp x6, x7, [sp, #-16]!\n" \
        "stp x4, x5, [sp, #-16]!\n" \
        "stp x2, x3, [sp, #-16]!\n" \
        "stp x0, x1, [sp, #-16]!\n");
    
    #define resume() \
    __asm volatile ( \
        "ldp x0, x1, [sp], #16\n" \
        "ldp x2, x3, [sp], #16\n" \
        "ldp x4, x5, [sp], #16\n" \
        "ldp x6, x7, [sp], #16\n" \
        "ldp x8, x9, [sp], #16\n" );
        
    #define call(b, value) \
        __asm volatile ("stp x8, x9, [sp, #-16]!\n"); \
        __asm volatile ("mov x12, %0\n" :: "r"(value)); \
        __asm volatile ("ldp x8, x9, [sp], #16\n"); \
        __asm volatile (#b " x12\n");
    
    
    __attribute__((__naked__)) static void hook_Objc_msgSend() {
    
        save()
        __asm volatile ("mov x2, lr\n");
        __asm volatile ("mov x3, x4\n");
        
        call(blr, &push_msgSend)
        resume()
        call(blr, orig_objc_msgSend)
        
        save()
        call(blr, &pop_msgSend)
        
        __asm volatile ("mov lr, x0\n");
        resume()
        __asm volatile ("ret\n");
    }

| 指标 |  |
| --- | --- |
| 卡顿反馈 | 高 |
| 采集精度 | 高 |
| 性能损耗 | 拦截后调用频次非常高，启动阶段可达`10w`次以上调用。高 |
| 实现成本 | 需要维护方法栈和优化拦截算法。高 |
| 结论 | 准确率很高的工具，但不适用于`Swift`代码 |



![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)

