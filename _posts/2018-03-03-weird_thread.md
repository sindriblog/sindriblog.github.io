---
title: 奇怪的GCD
date: 2018-03-03 08:00:00
tags:
- 多线程
---

多线程一直是我相当感兴趣的技术知识之一，个人尤其喜爱`GCD`这个轻量级的多线程解决方案，为了了解其实现，不厌其烦的翻阅`libdispatch`的源码。甚至因为太喜欢了，本来想要写这相应的源码解析系列文章，但害怕写的不好，于是除了开篇的类型介绍，也是草草了事，没了下文

恰好这几天好友出了几道有关`GCD`的题目，运行结果出于意料，仔细摸索后，发现苹果基于`libdispatch`做了一些有趣的修改工作，于是想将这两道题目分享出来。由于朋友提供的运行代码为`Swift`书写，在此我转换成等效的`OC`代码进行讲述。你如果了解了下面两个概念，会让后续的阅读更加容易：

- 同步与异步的概念
- 队列与线程的区别

## 被误解的概念
对于主线程和主队列，我们可能会有这么一个理解

> 主线程只会执行主队列的任务。同样，主队列只会在主线程上被执行

### 主线程只会执行主队列的任务
首先是主线程只会执行主队列的任务。在`iOS`中，只有主线程才拥有权限向渲染服务提交打包的图层树信息，完成图形的显示工作。而我们在`work queue`中提交的`UI`更新总是无效的，甚至导致崩溃发生。而由于主队列只有一条，其他的队列全部都是`work queue`，因此可以得出`主线程只会执行主队列的任务`这一结论。但是，有下面这么一段代码：

    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    
    dispatch_queue_set_specific(mainQueue, "key", "main", NULL);
    dispatch_sync(globalQueue, ^{
        BOOL res1 = [NSThread isMainThread];
        BOOL res2 = dispatch_get_specific("key") != NULL;
        
        NSLog(@"is main thread: %zd --- is main queue: %zd", res1, res2);
    });
    
根据正常逻辑的理解来说，这里的两个判断结果应该都是`NO`，但运行后，第一个判断为`YES`，后者为`NO`，输出说明了主线程此时执行了`work queue`的任务


#### dispatch_sync
上面的代码在换成`async`之后就会得到预期的判断结果，但在同步执行的情况下就会导致这个问题。在查找原因之前，借用`bestswifter`文章中的代码一用，首先`sync`的调用栈以及大致源码如下：

    dispatch_sync  
        └──dispatch_sync_f
            └──_dispatch_sync_f2
                └──_dispatch_sync_f_slow


    static void _dispatch_sync_f_slow(dispatch_queue_t dq, void *ctxt, dispatch_function_t func) {  
        _dispatch_thread_semaphore_t sema = _dispatch_get_thread_semaphore();
        struct dispatch_sync_slow_s {
            DISPATCH_CONTINUATION_HEADER(sync_slow);
        } dss = {
            .do_vtable = (void*)DISPATCH_OBJ_SYNC_SLOW_BIT,
            .dc_ctxt = (void*)sema,
        };
        _dispatch_queue_push(dq, (void *)&dss);

        _dispatch_thread_semaphore_wait(sema);
        _dispatch_put_thread_semaphore(sema);
        // ...
    }

可以看到对于`libdispatch`对于同步任务的处理是采用`sema`信号量的方式堵塞调用线程直到任务被处理完成，这也是为什么`sync`嵌套使用是一个死锁问题。根据源码可以得到执行的流程图：

![](http://p0zs066q3.bkt.clouddn.com/2018030301.jpg)

但实际运行后，`block`是执行在主线程上的，代码真正流程是这样的：

![](http://p0zs066q3.bkt.clouddn.com/2018030302.jpg)

因此可以做一个猜想：

> 由于`sync`函数本身会堵塞当前执行线程直到任务执行。为了减少线程切换的开销，以及避免线程被堵塞的资源浪费，于是对`sync`函数进行了改进：在大多数情况下，直接在当前线程执行同步任务

既然有了猜想，就需要验证。之所以说是大多数情况，是因为目前`主队列只在主线程上被执行`还是有效的，因此我们排除`global -sync-> main`这种条件。因此为了验证效果，需要创建一个串行线程：

    dispatch_queue_t serialQueue = dispatch_queue_create("serial.queue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    
    dispatch_sync(globalQueue, ^{
        BOOL res1 = [NSThread isMainThread];
        BOOL res2 = dispatch_get_specific("key") != NULL;
        
        NSLog(@"is main thread: %zd --- is main queue: %zd", res1, res2);
    });
    
    dispatch_async(globalQueue, ^{
        NSThread *globalThread = [NSThread currentThread];
        dispatch_sync(serialQueue, ^{
            BOOL res = [NSThread currentThread] == globalThread;
            NSLog(@"is same thread: %zd", res);
        });
    });
    
运行后，两次判断的结果都是`YES`，结果足以验证猜想，可以确定苹果为了提高性能，已经对`sync`做了修改。另外`global -sync-> main`测试结果发现`sync`的调用过程不会被优化

### 主队列只会在主线程上执行
上面说过，只有主线程才有权限提交渲染任务。同样的，出于下面两个设定，这个理解应当是成立的：

- 主队列总是可以调用`UIKit`的接口`api`
- 同时只有一条线程能够执行串行队列的任务

同样的，朋友给出了另一份代码：

    
    dispatch_queue_set_specific(mainQueue, "key", "main", NULL);
    
    dispatch_block_t log = ^{
        printf("main thread: %zd", [NSThread isMainThread]);
        void *value = dispatch_get_specific("key");
        printf("main queue: %zd", value != NULL);
    }
    
    dispatch_async(globalQueue, ^{
        dispatch_async(dispatch_get_main_queue(), log);
    });
    
    dispatch_main();
    
运行之后，输出结果分别为`NO`和`YES`，也就是说此时主队列的任务并没有在主线程上执行。要弄清楚这个问题的原因显然难度要比上一个问题难度大得多，因为如果子线程可以执行主队列的任务，那么此时是无法提交打包图层信息到渲染服务的

![](http://p0zs066q3.bkt.clouddn.com/2018030303.png)

同样的，我们可以先猜测原因。不同于正常的项目启动代码，这个`Swift`文件的运行更像是脚本运行，因为缺少了一段启动代码：

    @autoreleasepool
    {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }

为了找到答案，首先需要对问题`主线程只会执行主队列的任务`的代码进行改造一下。另外由于第二个问题涉及到`执行任务所在的线程`，`mach_thread_self`函数会返回当前线程的`id`，可以用来判断两个线程是否相同：
    
    thread_t threadId = mach_thread_self();
    
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    dispatch_queue_t serialQueue = dispatch_queue_create("serial.queue", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    
    dispatch_async(globalQueue, ^{
        dispatch_async(mainQueue, ^{
            NSLog(@"%zd --- %zd", threadId == mach_thread_self(), [NSThread isMainThread]);
        });
    });
    
    @autoreleasepool
    {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
    
这段代码的运行结果都是`YES`，说明在`UIApplicationMain`函数前后主队列任务执行的线程`id`是相同的，因此可以得出两个条件：

- 主队列的任务总是在同一个线程上执行
- 在`UIApplicationMain`函数调用后，`isMainThread`返回了正确结果

结合这两个条件，可以做出猜想：在`UIApplicationMain`中存在某个操作使得原本执行主队列任务的线程变成了`主线程`，其猜想图如下：

![](http://p0zs066q3.bkt.clouddn.com/2018030304.png)

由于`UIApplicationMain`是个私有`api`，我们没有其实现代码，但是我们都知道在这个函数调用之后，主线程的`runloop`会被启动，那么这个线程的变动是不是跟`runloop`的启动有关呢？为了验证这个判断，在手动启动`runloop`定时的去检测线程：
    
    dispatch_block_t log = ^{
        printf("is main thread: %zd\n", [NSThread isMainThread]);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), log);
    }
    
    dispatch_async(globalQueue, ^{
        dispatch_async(dispatch_get_main_queue(), log);
    });
    
    [[NSRunLoop currentRunLoop] run];
    
在`runloop`启动后，所有的检测结果都是`YES`：

    // console log
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"
    "is main thread: 1"

代码的运行结果验证了这个猜想，但结论就变成了：

> `thread` -> `runloop` -> `main thread`

这样的结论，随便启动一个`work queue`的`runloop`就能轻易的推翻这个结论，那么是否可能只有第一次启动`runloop`的线程才有可能变成主线程？为了验证这个猜想，继续改造代码：

    dispatch_queue_t serialQueue = dispatch_queue_create("serial.queue", DISPATCH_QUEUE_SERIAL);
    
    dispatch_block_t logSerial = ^{
        printf("is main thread: %zd\n", [NSThread isMainThread]);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), serialQueue, log);
    }
    
    dispatch_async(serialQueue, ^{
        [[NSRunLoop currentRunLoop] run];
    });
    dispatch_async(globalQueue, ^{
        dispatch_async(serialQueue, logSerial);
    });
    
    dispatch_main();
    
在保证了子线程的`runloop`是第一个被启动的情况下，所有运行的输出结果都是`NO`，也就是说因为`runloop`修改了线程的`priority`的猜想是不成立的，那么基于`UIApplicationMain`测试代码的两个条件无法解释`主队列为什么没有运行在主线程上`

#### 主队列不总是在同一个线程上执行
经过来回推敲，我发现`主队列总是在同一个线程上执行`这个条件限制了进一步扩大猜想的可能性，为了验证这个条件，通过定时输出主队列任务所在的`threadId`来检测这个条件是否成立：

    thread_t threadId = mach_thread_self();
    dispatch_queue_t serialQueue = dispatch_queue_create("serial.queue", DISPATCH_QUEUE_SERIAL);
    printf("current thread id is: %d\n", threadId);
    
    dispatch_block_t logMain = ^{
        printf("=====main queue======> thread id is: %d\n", mach_thread_self());
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), logMain);
    }
    
    dispatch_block_t logSerial = ^{
        printf("serial queue thread id is: %d\n", mach_thread_self());
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), serialQueue, logSerial);
    }
    
    dispatch_async(globalQueue, ^{
        dispatch_async(serialQueue, logSerial);
        dispatch_async(dispatch_get_main_queue(), logMain);
    });
    
    dispatch_main();
    
在测试代码中增加子队列定时做对比，发现不管是`serial queue`还是`main queue`，都有可能运行在不同的线程上面。但是如果去掉了子队列作为对比，`main queue`只会执行在一条线程上，但该线程的`threadId`总是不等同于我们保存下来的数值：
    
    // console log
    current thread id is: 775
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 7171"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 1547"
    "serial queue thread id is: 1547"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 1547"
    "serial queue thread id is: 1547"
    "=====main queue======> thread id is: 6403"
    "=====main queue======> thread id is: 1547"
    "serial queue thread id is: 6403"
    "serial queue thread id is: 4355"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 4355"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 4355"
    "=====main queue======> thread id is: 4355"
    "serial queue thread id is: 6403"
    "serial queue thread id is: 1547"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 1547"
    "=====main queue======> thread id is: 6403"
    "serial queue thread id is: 1547"
    "serial queue thread id is: 6403"
    "=====main queue======> thread id is: 1547"
    "serial queue thread id is: 1547"
    
发现了这一个新的现象后，结合之前的信息来看，可以得出一个新的猜想：

> 有一个专用启动线程用于启动主线程的`runloop`，启动前主队列会被这个线程执行

要测试这个猜想也很简单，只要对比`runloop`前后的`threadId`是否一致就可以了：

    thread_t threadId = mach_thread_self();
    printf("current thread id is: %d\n", threadId);
    
    dispatch_block_t logMain = ^{
        printf("=====main queue======> thread id is: %d\n", mach_thread_self());
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), logMain);
    }
    
    dispatch_block_t logSerial = ^{
        printf("serial queue thread id is: %d\n", mach_thread_self());
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), serialQueue, logSerial);
    }
    
    dispatch_async(globalQueue, ^{
        dispatch_async(serialQueue, logSerial);
        dispatch_async(dispatch_get_main_queue(), logMain);
    });
    
    [[NSRunLoop currentRunLoop] run];
    
    // console log
    current thread id is: 775
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    "=====main queue======> thread id is: 775"
    
运行结果说明了并不存在什么`启动线程`，一旦`runloop`启动后，主队列就会一直执行在同一个线程上，而这个线程就是主线程。由于`runloop`本身是一个不断循环处理事件的死循环，这才是它启动后主队列一直运行在一个主线程上的原因。最后为了测试启动`runloop`对串行队列的影响，单独启动子队列和一起启动后，发现另一个现象：

- 主队列的`runloop`一旦启动，就只会被该线程执行任务
- 子队列的`runloop`无法绑定队列和线程的执行关系

由于在源码中`async`调用对于主队列和子队列的表现不同，后者会直接启用一个线程来执行子队列的任务，这就是导致了`runloop`在主队列和子队列上差异化的原因，也能说明苹果并没有大肆修改`libdispatch`的源码

## 有趣的runloop唤醒机制
如果你看过`runloop`相关的博客或者文档，那么应该会它是一个不断处理消息、事件的死循环，但死循环是会消耗大量的`cpu`资源的（自旋锁就是死循环空转）。`runloop`为了提高线程的使用效率以及减少不必要的损耗，在没有事件处理的时候，假如此时存在`timer、port、source`任一一种，那么进入休眠状态；假如不存在三者其中之一，那么`runloop`将会退出

![](http://p0zs066q3.bkt.clouddn.com/2018030305.png)

因此为了探讨`runloop`的唤醒，我们可以通过添加一个空端口来维持`runloop`的运转：

    CFRunLoopRef runloop = NULL;
    NSThread *thread = [[NSThread alloc] initWithBlock: ^{
        runloop = [NSRunLoop currentRunLoop].getCFRunLoop;
        [[NSRunLoop currentRunLoop] addPort: [NSMachPort new] forMode: NSRunLoopCommonModes];
        [[NSRunLoop currentRunLoop] run];
    }];
    
这里主要讨论的是仓鼠大佬的第五题，原问题可以直接到最下面翻链接。主要要说明的是问题中提到的两个`api`，用于添加任务到这个`runloop`中：

    CFRunLoopPerformBlock(runloop, NSRunLoopCommonModes, ^{
        NSLog(@"runloop perform block 1");
    });
    
    [NSObject performSelector: @selector(log) onThread: thread withObject: obj waitUntilDone: NO];
    
    CFRunLoopPerformBlock(runloop, NSRunLoopCommonModes, ^{
        NSLog(@"runloop perform block 2");
    });
    
上面的代码如果去掉了第二个`perform`调用，那么第一个调用不会输出，反之就会都输出。从名字上看，两个调用都是往所在的线程里面添加执行任务，区别在于后者的调用实际上并不是直接插入任务`block`，而是将任务包装成一个`timer`事件来添加，这个事件会唤醒`runloop`。当然，前提是`runloop`处在休眠中。

> `CFRunLoopPerformBlock`提供了往`runloop`中添加任务的功能，但又不会唤醒`runloop`，在事件很少的情况下，这个`api`能有效的减少线程状态切换的开销

## 其他
过了一个漫长的春节假期之后，感觉急需一个节假日来休息，可惜这只是奢望。由于节后综合征，在这周重新返工的状态感觉一般，也偶尔会提不起神来，希望自己尽快恢复过来。另外随着不断的积累，一些自以为熟悉的奇怪问题又总能带来新的认知和收获，我想这就是学习最大的快乐了

## 关于使用代码
由于`Swift`语法上和`OC`可能存在差异，建议读者可以阅读`Swift`源码对比本文讲述。关注下方`仓鼠大佬`的博客链接，大佬放话后续会放出源码。另外如果不想阅读`libdispatch`源码又想对这部分的逻辑有所了解的朋友可以看下面的链接文章

## 扩展阅读
[仓鼠大佬](https://www.jianshu.com/subscriptions#/subscriptions/227486/user)

[深入了解GCD](https://bestswifter.com/deep-gcd/)

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)

