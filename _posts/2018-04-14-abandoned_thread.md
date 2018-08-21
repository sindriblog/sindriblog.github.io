---
title: 被遗弃的线程
date: 2018-04-14 08:00:00
categories:
- Note
tags:
- Note
- Tips
- 多线程
---

`main`函数作为程序运行的入口，正常情况下，函数会执行毫秒级别的操作，然后返回一个`0`表示程序正常终止。为了避免应用启动即终止，苹果设计了`runloop`机制来维持线程的生命，`runloop`在每一次的循环当中不断的去处理事件，或控制线程的休眠和唤醒。`runloop`还结合了`libdispatch`的任务派发机制，可以循环地处理`async`到队列中的任务

## 启动runloop
从`runloop`对外暴露的接口来看，启动方式一共存在三种：

- 无条件的启动。这种启动方式缺少停止`runloop`的手段，唯一的结束方式是`kill`掉线程
    
        - (void)run;
    
- 设定超时时间。直接`run`会调用这个接口并且传入`distantFuture`，表示无限长的超时时间。如果超时时间不是无限长，那么`runloop`会在处理完事件或者超时后终止，优于直接`run`

        - (void)runUntilDate: (NSDate *)limitDate;

- 设置超时时间和运行模式。比起第二种，允许我们让`runloop`运行在某个模式下，灵活性更高

        - (BOOL)runMode: (NSRunLoopMode)mode beforeDate: (NSDate *)limitDate;

如果`runloop`在启动之后没有任何`sources`、`timers`或者`ports`事件可以处理，那么会自动退出，否则会在处理完成后让线程陷入休眠，等待这些事件重新唤醒线程处理。下面是最常用来表示`runloop`处理逻辑的示意图：

![](http://p0zs066q3.bkt.clouddn.com/2018041301.png)

除开图中列出的事件之外，`main loop`会处理`timer`之后检测队列中是否存在待执行的`block`然后开始执行

## 什么情况下需要启动runloop
主线程的`runloop`会在应用启动后被`UIApplication`启动，其他线程则需要我们主动去`run`。从接触`iOS`开发到现在，笔者了解的需要主动启动`runloop`只有这么两类：

- 子线程使用`timer`

    由于`NSTimer`本身就不是一个能确保稳定回调的定时器机制，并且主线程会经常处在忙碌状态，这又进一步降低了`NSTimer`的准确性。为了提高定时的准确性，多数人会采用子线程启动`runloop`的方式来实现定时器功能

- 线程保活
    
    线程保活是一种不太常见的需求，但是如果你曾经了解过`AFNetworking`的做法，会发现创建了子线程之后，采用一个空`port`的方式来启动`runloop`，避免线程被中止回收。但实际上这种做法很容易导致线程既无法被回收，也不能被使用的情况
    
上面两种不同的应用场景，实际上是使用`timer`和`port`维持`runloop`不会因为没有事件处理直接退出，而且在这些源事件来临之前，线程大多数情况下处在休眠状态不造成额外损耗

## 串行队列的runloop
假设现在需要使用一个子线程的`runloop`来实现定时器，由于`runloop`在停止之前，线程会一直存活，因此可能会想利用这个存活的线程处理其他的任务。因此除了`NSTimer`之外，我们添加一个`GCD Timer`定时的派发任务给这个启动`runloop`的队列：

    dispatch_queue_t serialQueue = dispatch_queue_create("serial.queue", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(serialQueue, ^{
        NSLog(@"the task run in the thread: %d", mach_thread_self());
        [NSTimer scheduledTimerWithTimeInterval: 0.5 repeats: YES block: ^(NSTimer * _Nonnull timer) {
            NSLog(@"ns timer in the thread: %d", mach_thread_self());
        }];
        [[NSRunLoop currentRunLoop] runUntilDate: [NSDate dateWithTimeIntervalSinceNow: 600]];
    });
    
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, serialQueue);
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.5 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        NSLog(@"gcd timer in the thread: %d", mach_thread_self());
    });
    dispatch_resume(timer);

按照预期，这段代码充分利用了已经被保活的线程，除了已有的`NSTimer`之外，线程还能在空闲的时间去处理不断派发的任务，但实际上只有`NSTimer`的任务被执行：

    2018-04-14 10:32:54.667718+0800 PThreads[7693:94702] the task run in the thread: 4355
    2018-04-14 10:32:55.168871+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:55.672011+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:56.169150+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:56.669411+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:57.169665+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:57.669234+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:58.172068+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:58.669446+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:59.169223+0800 PThreads[7693:94702] ns timer in the thread: 4355
    2018-04-14 10:32:59.671802+0800 PThreads[7693:94702] ns timer in the thread: 4355

导致保活线程无法处理`async`任务的原因有两个：

- `runloop`和`queue`的区别
    
    `runloop`和`queue`各自维护着自己的一个任务队列，在`runloop`的每个周期里面，会检测自身的任务队列里面是否存在待执行的`task`并且执行。但主线程的情况比较特殊，在`main runloop`的每个周期，会去检测`main queue`是否存在待执行任务，如果存在，那么`copy`到自身的任务队列中执行

- `async`的实现不同

    在非主线程之外，`runloop`和`queue`的任务队列是互不干扰的，因此两者处理任务的机制也是完全不同的。当`async`任务到队列时，`GCD`会尝试寻找一个线程来执行任务。由于串行队列同时只能与一个线程挂钩，因此`GCD`会让该线程执行完已有任务后，才执行`async`到队列中的任务。但由于线程被保活，任务是一个条件死循环`condition-loop`，因此`async`的任务始终无法被处理
    
为了证明这些原因，可以通过`CFRunLoopPerformBlock`将任务直接加入到`runloop`自身的任务队列中，检测这个任务是否被执行：

    __block CFRunLoopRef serialRunLoop = NULL;
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    dispatch_queue_t serialQueue = dispatch_queue_create("serial.queue", DISPATCH_QUEUE_SERIAL);
    
    dispatch_async(serialQueue, ^{
        NSLog(@"the task run in the thread: %d", mach_thread_self());
        [NSTimer scheduledTimerWithTimeInterval: 0.5 repeats: YES block: ^(NSTimer * _Nonnull timer) {
            NSLog(@"ns timer in the thread: %d", mach_thread_self());
        }];
        serialRunLoop = [NSRunLoop currentRunLoop].getCFRunLoop;
        [[NSRunLoop currentRunLoop] runUntilDate: [NSDate dateWithTimeIntervalSinceNow: 600]];
    });
    
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, mainQueue);
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.5 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        dispatch_async(serialQueue, ^{
            NSLog(@"gcd timer in the thread: %d", mach_thread_self());
        });
        CFRunLoopPerformBlock(serialRunLoop, NSDefaultRunLoopMode, ^{
            NSLog(@"perform block in thread: %d", mach_thread_self());
        });
    });
    dispatch_resume(timer);

再次运行之后，`async`的任务依旧无法被处理，但是`perform block`的任务总是能在`timer`唤醒休眠的线程后被处理：

    2018-04-14 11:01:29.925924+0800 PThreads[15619:198121] the task run in the thread: 4355
    2018-04-14 11:01:30.428175+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:30.428982+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:30.429410+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:30.932411+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:30.932674+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:31.430281+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:31.430546+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:31.929485+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:31.929691+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:32.432369+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:32.432726+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:32.930981+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:32.931179+0800 PThreads[15619:198121] perform block in thread: 4355
    2018-04-14 11:01:33.429207+0800 PThreads[15619:198121] ns timer in the thread: 4355
    2018-04-14 11:01:33.429519+0800 PThreads[15619:198121] perform block in thread: 4355

## 并行队列的runloop
队列的`串并行`属性决定了队列能不能被多个线程处理任务，因此同样的代码在并行队列执行，产生的结果必然是有所区别的：

    __block CFRunLoopRef serialRunLoop = NULL;
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    dispatch_queue_t serialQueue = dispatch_queue_create("concurrent.queue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(serialQueue, ^{
        NSLog(@"the task run in the thread: %d", mach_thread_self());
        [NSTimer scheduledTimerWithTimeInterval: 0.5 repeats: YES block: ^(NSTimer * _Nonnull timer) {
            NSLog(@"ns timer in the thread: %d", mach_thread_self());
        }];
        serialRunLoop = [NSRunLoop currentRunLoop].getCFRunLoop;
        [[NSRunLoop currentRunLoop] runUntilDate: [NSDate dateWithTimeIntervalSinceNow: 600]];
    });
    
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, mainQueue);
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.5 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        dispatch_async(serialQueue, ^{
            NSLog(@"gcd timer in the thread: %d", mach_thread_self());
        });
        CFRunLoopPerformBlock(serialRunLoop, NSDefaultRunLoopMode, ^{
            NSLog(@"perform block in thread: %d", mach_thread_self());
        });
    });
    dispatch_resume(timer);
    
输出结果如下：
    
    2018-04-14 11:02:47.348083+0800 PThreads[15988:203200] the task run in the thread: 4867
    2018-04-14 11:02:47.397174+0800 PThreads[15988:203203] gcd timer in the thread: 3843
    2018-04-14 11:02:47.840678+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:47.852870+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:47.853122+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:47.853552+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:48.340602+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:48.352863+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:48.353149+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:48.840085+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:48.853918+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:48.854120+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:49.340729+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:49.354172+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:49.354470+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:49.840661+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:49.853115+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:49.853288+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:50.340078+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:50.354262+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:50.354537+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:50.840653+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:50.854117+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:50.854406+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:51.339972+0800 PThreads[15988:203206] gcd timer in the thread: 9219
    2018-04-14 11:02:51.353246+0800 PThreads[15988:203200] ns timer in the thread: 4867
    2018-04-14 11:02:51.353472+0800 PThreads[15988:203200] perform block in thread: 4867
    2018-04-14 11:02:51.839917+0800 PThreads[15988:203206] gcd timer in the thread: 9219

虽然并行队列的`async`功能并不会因为启动了`runloop`受到影响，但是可以发现如果不去保存`runloop`，这个保活的线程除了定时器能正常处理之外，其他时候不会再被`GCD`复用

## 使用port保活
如果不使用`NSTimer`这种稳定的唤醒机制来保活线程，而是采用`port`的方式，线程的表现是否依旧符合预期？

    __block CFRunLoopRef serialRunLoop = NULL;
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    dispatch_queue_t serialQueue = dispatch_queue_create("concurrent.queue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(serialQueue, ^{
        serialRunLoop = [NSRunLoop currentRunLoop].getCFRunLoop;
        [[NSRunLoop currentRunLoop] addPort: [NSPort new] forMode: NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] runUntilDate: [NSDate dateWithTimeIntervalSinceNow: 600]];
    });
    
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, mainQueue);
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.5 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        CFRunLoopPerformBlock(serialRunLoop, NSDefaultRunLoopMode, ^{
            NSLog(@"perform block in thread: %d", mach_thread_self());
        });
    });
    dispatch_resume(timer);
    
此时任务总是不会被处理。由于`runloop`需要被唤醒才能处理队列任务，而`perform block`只是单纯的添加任务，没有唤醒功能。为了线程能够继续执行任务，这时候还需要不断的`wake up`线程：

    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, mainQueue);
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.5 * NSEC_PER_SEC, 0 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        CFRunLoopPerformBlock(serialRunLoop, NSDefaultRunLoopMode, ^{
            NSLog(@"perform block in thread: %d", mach_thread_self());
        });
        CFRunLoopWakeUp(serialRunLoop);
    });
    dispatch_resume(timer);

    /// 日志输出
    2018-04-14 11:15:10.314064+0800 PThreads[19459:248764] perform block in thread: 2563
    2018-04-14 11:15:10.763480+0800 PThreads[19459:248764] perform block in thread: 2563
    2018-04-14 11:15:11.263651+0800 PThreads[19459:248764] perform block in thread: 2563
    2018-04-14 11:15:11.763957+0800 PThreads[19459:248764] perform block in thread: 2563
    2018-04-14 11:15:12.264006+0800 PThreads[19459:248764] perform block in thread: 2563
    2018-04-14 11:15:12.763384+0800 PThreads[19459:248764] perform block in thread: 2563
    
## 结论
从测试来看，在子线程启动`runloop`并不是一个很明智的选择：这会导致线程保活期间被遗弃，失去了处理消息派发的能力，且无法响应其他线程的通信。其次，即便可以通过`perform block`来继续为保活线程添加任务处理，但在保活线程的`runloop`缺乏稳定的唤醒机制的情况下，还需要其他线程来提供唤醒能力，这增加了代码设计的成本，并且不会有额外的好处

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)


