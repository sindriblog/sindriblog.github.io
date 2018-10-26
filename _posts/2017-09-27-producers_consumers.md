---
title: 多线程-生产者消费者
date: 2017-09-27 08:00:00
tags:
- 多线程
---

在计算机世界当中，数据在不断产生的同时，也在不停地被处理着。产生数据的一方被我们称作`生产者`，而另一方则被称为`消费者`。在网络请求中，`服务器`是提供数据的`生产者`，而`用户`是数据的`消费者`。伴随着互联网的高速发展，即便是百万乃至千万用户级的访问量也并非罕见，而`服务器`在面临如此高的并发请求时仍能做到游刃有余。这背后的技术并不是三言两语能够说清的，但其中关系着一个重要的模式——`生产者-消费者模式`

> `生产者-消费者模式`是通过增加缓冲区来平衡`生产者`和`消费者`的速率差问题。`生产者`和`消费者`彼此不直接通信，通过缓冲区来完成任务

打个比方，以前肯德基的点餐员在点餐之后会顾客配好餐，然后再结账。这种时候，后排的人总是要等到前排完成所有的环节之后才能开始点餐，而现在用户点完餐只需要等待叫号取餐。`取餐区`就是肯德基在就餐环节中加入的缓冲区

> 好的模式总是源于生活

`生产者-消费者模式`似乎更多的被应用在后端的并发处理中，前端开发也很少去提及这一模式。其实在开发的各个环节中，系统早就做了很多`生产者-消费者`的平衡工作，我们在享受着它带来的便利的同时，却不自知。本文盘点在开发一些常见的`生产者-消费者`

## 线程&队列
由于`生产者-消费者模式`是为了平衡二者因处理能力的不同而产生的速率差问题，所以速率差具体表现存在两种情况：`生产>消费`和`生产<消费`：

- `生产>消费`

    比较常见的一种情况是存在多个并发的网络请求，当请求任务执行完毕时，需要将数据持久化到数据库中。为了避免数据被破坏，在网络层和数据层中间一般会引入一个同步处理机制，作为平衡数据生产和数据处理的缓冲区。在应用中，同步处理机制可能会在网络层中完成，也可能放到数据层当中
        
    ![](http://upload-images.jianshu.io/upload_images/783864-d809ba515416c31f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    前者在网络请求层创建了一个串行队列，请求结果`dispatch`到这个队列中进行回调，甚至直接`dispatch`到主线程中回调即可。第三方网络请求库如`AFNetworking`就在在回调前将请求结果处理放在同一个队列中执行
 
        [session dataTaskWithRequest: request 
                    completionHandler: ^(NSData * _Nullable data, 
                                        NSURLResponse * _Nullable response, 
                                        NSError * _Nullable error) {
            if (error != nil) {
                dispatch_async(dispatch_get_main_queue(), ^ {
                    complete(data, error);
                });
            }
        }
 
    后者则是在数据层内部创建了一个串行队列，将所有`IO`操作都派发到这个队列中执行
 
        - (void)insertData: (NSData *)data {
            dispatch_async(_dbQueue, ^{
                [_db executeUpdate: [self _insertSqlWithData: data];
            });
        }
    
- `消费>生产`
    
    在派发任务少于可执行线程数量时，`数据消费`的能力要高于`数据产出`。在平时开发中通过`async`将任务派发到并发队列之后，`GCD`会从缓存的线程池查找可执行线程，然后执行任务。在[libdispatch](https://github.com/nickhutchinson/libdispatch)中可以看到`GCD`线程池最多缓存`64`个线程
     
    ![](http://upload-images.jianshu.io/upload_images/783864-a62271f729c63c24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    考虑到并发队列的可控性太弱，如果对任务执行顺序有需求，又想享受`GCD`动态控制线程带来的好处，那么维护一套自己的`serialQueue`或者使用`NSOperationQueue`也是可行的，下面代码截取自[GCD封装](http://sindrilin.com/note/2017/04/06/GCD封装/)
 
        LXD_INLINE DispatchContext __LXDDispatchContextGetForQos(LXDQualityOfService qos) {
            static DispatchContext contexts[5];
            int count = (int)[NSProcessInfo processInfo].activeProcessorCount;
        count = MIN(1, MAX(count, LXD_QUEUE_MAX_COUNT));
            switch (qos) {
                case LXDQualityOfServiceUserInteractive: {
                    static dispatch_once_t once;
                    dispatch_once(&once, ^{
                        contexts[0] = __LXDDispatchContextCreate("com.sindrilin.user_interactive", count, qos);
                    });
                    return contexts[0];
                }
            
                case LXDQualityOfServiceUserInitiated: {
                    static dispatch_once_t once;
                    dispatch_once(&once, ^{
                        contexts[1] = __LXDDispatchContextCreate("com.sindrilin.user_initated", count, qos);
                    });
                    return contexts[1];
                }
            
                case LXDQualityOfServiceUtility: {
                    static dispatch_once_t once;
                    dispatch_once(&once, ^{
                        contexts[2] = __LXDDispatchContextCreate("com.sindrilin.utility", count, qos);
                    });
                    return contexts[2];
                }
            
                case LXDQualityOfServiceBackground: {
                    static dispatch_once_t once;
                    dispatch_once(&once, ^{
                        contexts[3] = __LXDDispatchContextCreate("com.sindrilin.background", count, qos);
                    });
                    return contexts[3];
                }
            
                case LXDQualityOfServiceDefault:
                default: {
                    static dispatch_once_t once;
                    dispatch_once(&once, ^{
                        contexts[4] = __LXDDispatchContextCreate("com.sindrilin.default", count, qos);
                });
                    return contexts[4];
                }
            }
        }
    
    当然，除了`任务队列`方案以外，`pthread_t`提供了创建、维护线程的能力，如果有需要，我们也能维护自己的线程池来获得更好的控制能力，可以参见[EvansMusic的回答](https://zhuanlan.zhihu.com/p/22834934)的回答，下面是使用线程池的伪代码
 
        if (_threadsPool.count > 0) {
            Thread *thread = _threadsPool.pop();
            thread.wakeup();
            thread.execute(task);
        } else {
            Thread *thread = Thread();
            thread.execute(task);
            thread_sleep(thread);
            _threadPool.append(thread);
        }

## 锁&信号量
当锁作为缓冲机制生效时，基本是`消费>生产`的情况。通过线程锁让`消费者`即`线程`陷入休眠中避免造成不必要的开销，等待唤醒`消费`资源。

    if (_lock.try()) {
        thread.consume(resource);
    } else {
        thread.sleep();
        wait_threads.append(thread);
    }

`信号量`是多线程中功能更强大的`缓冲机制`，它允许大于`1`个的`消费者`进入临界区：

![](http://upload-images.jianshu.io/upload_images/783864-42fb7e6a7871ef63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`GCD`的信号量实现可以参见[Semaphore原理与实现](http://www.jianshu.com/p/947153c6b409)

## 双缓冲机制
`双缓冲机制`在笔者的印象中在硬件上的实现居多，软件上见到的相对比较少，比如`GCD`就采用了双缓存区的方式。虽然`iOS`允许我们创建多个任务队列、并设置不同的优先级，但是任务最终会根据优先级和串行是否并行重新加入到系统的八个全局队列中。`overcommit`属性的队列表示的是并行执行的任务队列，在有新任务加入进来之后如果没有可执行的线程，那么就会新建线程去执行这个任务

![](http://upload-images.jianshu.io/upload_images/783864-cf365326938674a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

除了`GCD`的任务队列设计，`iOS`的渲染机制也是采用双缓冲区的方式实现的。双缓冲区相较单缓冲区更好的处理了帧数据的读取和刷新的效率平衡，但是也可能导致页面撕裂的情况。为了解决这个问题，`iOS`始终使用垂直同步，详细可以参见[iOS保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
 
## 其他
`消费者-生产者`是一种经过漫长历史考验的代码设计之一，`缓冲区`的加入很好的避免了两者需要等待对方执行完成，很好的平衡了二者速率不一导致的资源浪费。也顺带也解除了两者的依赖，实现了解耦

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)


