---
title: 分析实现-倒计时设计
date: 2018-01-06 08:00:00
tags:
- 分析实现
---

计算机是不存在倒计时这个概念的，所有的倒计时设计来源于对定时器的运用：给予一个`deadline`，以秒为时间间隔，在每次回调时刷新屏幕上的数字。倒计时的实现几乎没有门槛，无论`NSTimer`也好，`GCD`也罢，甚至使用`CADisplayLink`都能用来制作一个倒计时方案。但同样的，低门槛也意味着方案的上下限差也很大，本文打算谈谈如何设计一个倒计时方案

### 为什么要写这篇文章
事实上，倒计时和我目前开发的方向八竿子打不着，我也确实没有必要和想过写这么一套方案。只是这几天有朋友分享了别人设计的倒计时功能：

> 采用一个全局计时管理对象针对每一个倒计时按钮分配计时器，计时器会生成一个`NSOperation`对象来执行回调，完成倒计时功能

在抛开代码不谈的情况下，这套设计思路我也是存疑的。如果倒计时要使用`operation`，那就需要使用`queue`来完成任务。根据`queue`的串行并行属性，要考虑这两点：

- 如果`queue`是并行的，一个界面上存在多个倒计时按钮时，可能会新建线程来处理同一个`queue`的任务，这部分的开销并不是必需的

- `operation`需要投放到`queue`里面启动执行。假如每秒的回调包装成`operation`处理，那么需要一个定时机制来投放这些`operation`。如果是这么，为什么不直接使用定时器，而要用`operation`

但在看完设计者的文章和代码之后，我发现对方根本没有考虑过上面的问题。他`operation`的任务思路很奇怪：

> 在每一个`operation`里面，采用`while + sleep`的方式，每次回调后让线程睡眠一秒，直至倒计时结束

    - (void)main {
        ......
        do {
            callback(self.leftTime);
            [NSThread sleepForTimeInterval: 1];
        } while (--self.leftTime > 0);
        ......
    }

这种实现有三个坑爹的地方：

1. `while`循环结束之前，内部的临时变量不会被释放，存在内存占用过大的风险

2. 如果`queue`是串行属性，多个`operation`将无法保证回调时间的正确

3. 不应该采用`sleep`方式计时，这很浪费线程的执行效率

另外，应用进入后台时，所有的子线程会被停止执行任务，这个会导致了应用切换前后台后，倒计时剩余时间不准。对于这种情况一般也有三种方式来做时间校正：

1. 保存一个倒计时`deadline`，在进入`active`后重新计算剩余倒计时

2. 注册通知，在切换前后台时计算时长，减去这个时间更新剩余时间

3. 创建应用后台任务，继续进行倒计时

而上面的设计采用了`3`的解决方案，鉴于应用在后台时，用户对应用毫无感知的特点，这几乎是最差的一种方案。于是基于这一个个槽点，我决定按照自己的想法，做一个相对高效的倒计时方案

### 存储结构
一套功能方案设计的目的是为了`简化逻辑流程，隐藏实现细节，尽可能少的暴露接口`。普通的倒计时是调用方直接使用定时器实现规律性回调，定时器需要持有调用方的信息。而倒计时设计隐藏了定时器的实现细节，只需调用方提供回调，其余的耦合关联由管理类来完成，类似的设计有`Notification`和`Observer`等

![](http://p0zs066q3.bkt.clouddn.com/2018010616.png)

![](http://p0zs066q3.bkt.clouddn.com/2018010610.png)

即便是系统使用的两种类似的监听设计，在内部实现时，所用到的存储结构也是不同的。`Notification`不持有对象本身，采用保存对象地址的实现，但这样存在野指针风险。`Observer`会持有对象，但会造成循环引用的风险。可以说：

> 不同的考量标准和业务场景决定了不同的结构设计

#### 回调设计
在`iOS`中常用的回调方式包括`delegate`和`block`两种，这两种方式都能很好的处理倒计时的需求。但出于以下理由我选择了`block`：

- `delegate`的耦合关系强于`block`

    `delegate`在委托方和代理方中间添加了一层中间层，解除了双方的直接耦合关系，但同样的委托方和代理方需要依赖于`protocol`，这是这种模式必然存在的耦合关系。相比之下，`block`并不存在这种烦恼
    
    ![](http://p0zs066q3.bkt.clouddn.com/2018010611.png)
    
- 更少的代码

    同样的回调处理下，`delegate`的代码量要多于`block`。并不是说更多的代码一定不好，只是同样符合实现需求，逻辑清晰同样清晰的情况下，后者优于前者：
    
        #pragma mark - delegate type
        - (void)doSomething {
            /// do something
            if ([_delegate respondsToSelector: @selector(countDownWithLeftTime:)]) {
                [_delegate countDownWithLeftTime: leftTime];
            }
        }
        
        #pragma mark - block type
        - (void)doSomething: (void(^)(NSInteger leftTime))countDown {
            /// do something
            if (countDown) { countDown(leftTime); }
        }
        
#### 任务中止
倒计时设计应当满足最基本的停止条件：`倒计时归零`，但除了正常中止之外，总是有提前结束任务的可能性。对外界提供`removeXXX`格式的接口是一种好的思路，这让代码成对匹配，赏心悦目。但在实际开发中，我们应当重视一个设计原则：

> 接口隔离原则：一个类与另一个类之间的依赖性，应该依赖于尽可能小的接口

对外暴露接口需要再三思考，因为一旦提供了额外的接口，这意味着实现方要多处理一个逻辑，调用方需要多调用一次代码。决定是否提供`remove`接口应该取决于对象依赖，`Notification`和`Observer`都会因为保存对象的关键信息导致额外的风险，因此必须提供移除策略

由于我们已经采用`block`设计，无需直接依赖对象的某些信息，可以忽略`remove`接口的选项。而实际上，想要提供中止任务的功能，系统的`enum`为我们提供了很好的思路：在回调中提供标记位，一旦标记位被修改，回调完成后中止任务。结合回调设计采用的方案，得到`block`的结构：

    typedef void(^LXDReceiverCallback)(long leftTime, bool *isStop);

#### 重复任务
通知中心是去重实现中很典型的`bad case`：同一次通知的回调次数取决于注册该通知的次数。如果将这种设计放到倒计时中，具体表现为剩余时长以正常倒计时数倍的速率减少。正常场景下很难出现这种问题，但如果在限时抢购的列表界面，多个`cell`被复用的情况下，很容易出现这种情况

    typedef struct hash_entry_t {
        void *entry;
    } hash_entry_t;
    
    class LXDBaseHashmap {
    public:
        LXDBaseHashmap();
        virtual ~LXDBaseHashmap();
        unsigned int entries_count;
        hash_entry_t *hash_entries;
        
    protected:
        unsigned int obj_hash_code(void *obj);
    };

`hash表`是一种常用的去重算法：一个输入只对应一个输出。为了避免对象重复创建倒计时任务，我们需要找到对象所拥有的一个唯一值作为输入——对象地址，并且以此作为`key`检测是否存在冲突：

    #define hash_bucket_count 7

    unsigned int LXDBaseHashmap::obj_hash_code(void *obj) {
        uint64_t *val1 = (uint64_t *)obj;
        uint64_t *val2 = val1 + 1;
        return (unsigned int)(*val1 + *val2) % hash_bucket_count;
    }

`hash表`需要考量的另一个因素是表的长度。同样情况下，表长度越大，越难发生计算冲突，但同样也占用了更多的内存。出于以下原因，最终桶的长度设定为`7`：

1. 倒计时场景下出现大量倒计时的可能性低
2. 由于地址都是双数，桶的个数必须为单数，否则无法计算出单数结果

另一方面，由于`hash表`的长度仅为`7`，出现计算冲突的并不小，我们还要考虑出现冲突时的处理方案，可以借鉴`@synchronized`的实现方式来解决这个问题。`@synchronized`同样基于对象地址进行`hash`计算，每个桶存储一张可用锁链表，链表的节点保存对象，用于执行冲突后的再匹配工作：

    typedef struct SyncData {
        id object;
        recursive_mutex_t mutex;
        struct SyncData* nextData;
        int threadCount;
    } SyncData;
    
    typedef struct SyncList {
        SyncData *data;
        spinlock_t lock;
    } SyncList;

    #define COUNT 16
    #define HASH(obj) ((((uintptr_t)(obj)) >> 5) & (COUNT - 1))
    #define LOCK_FOR_OBJ(obj) sDataLists[HASH(obj)].lock
    #define LIST_FOR_OBJ(obj) sDataLists[HASH(obj)].data
    static SyncList sDataLists[COUNT];
    
因此，仿照这种设计，我们存储的链表节点除了保存回调`block`和倒计时长之外，还需要保存调用对象的地址信息，得到结构体：

    typedef struct LXDReceiver {
        long lefttime;
        uintptr_t objaddr;
        LXDReceiverCallback callback;
    } LXDReceiver;

#### 链表性能
由于链表的查找需要遍历链表，效率较低。在倒计时完毕或者任务中止时，移除任务会有不必要的损耗。采用`双向链表`的设计，可以获取到删除任务的前后节点，快速的完成删除。另外预留一个链表长度变量，每个链表提供一个表头来提供长度属性，方便后续扩充：

    typedef struct LXDReceiverNode {
        unsigned int count;
        LXDReceiver *receiver;
        LXDReceiverNode *next;
        LXDReceiverNode *previous;
        
        LXDReceiverNode(LXDReceiver *receiver = NULL, LXDReceiverNode *previous = NULL) {
            this->count = 0;
            this->next = NULL;
            this->previous = previous;
            this->receiver = receiver;
        }
    } LXDReceiverNode;

至此为止，方案的数据结构已经确定，剩下的就是对倒计时逻辑的封装设计

### 逻辑封装

#### 定时器方案
常用于实现倒计时的定时器有`NSTimer`和`GCD`两种，处于两点考虑，我选择了后者：

1. `NSTimer`需要启动子线程的`runloop`，另外`iOS10+`系统必须手动启用一次`runloop`才能完成定时器的移除

2. `GCD`具有更高的效率和精确度

除此之外，定时器的设计一般被分为`多定时器设计`和`单定时器设计`，两种各有优劣


- `多定时器设计`
    
    ![](http://p0zs066q3.bkt.clouddn.com/2018010608.png)
    
    多定时器的设计下，每个倒计时任务拥有自己的计时器。优点在于可以单独控制每个任务的回调间隔。缺点是由于多个定时器的屏幕刷新不一定会同步，导致`UI`更新不同步等
    
- `单定时器设计`

    ![](http://p0zs066q3.bkt.clouddn.com/2018010602.png)
    
    单定时器的设计下，所有倒计时任务使用同一个计时器。优点在于减少了额外的性能损耗，设计结构更清晰。缺点在定时器已经启动的情况下，新任务的首次倒计时可能会有明显的提前以及多个倒计时任务强制使用同一种计时间隔。
    
考虑到倒计时的的`UI`同步效果以及更好的性能，我选择`单定时器设计`方案。另外如果确实存在多个不同计时间隔的需求，`单定时器设计`也可以很好的扩充接口提供支持

#### 注册任务
![](http://p0zs066q3.bkt.clouddn.com/2018010613.png)

对象在注册倒计时任务时，取对象的地址进行`hash计算`，根据结果找到存储的对象链表。链表节点存储的`objcaddr`用来匹配对象，如果匹配失败，那么新建一个任务回调节点。完成插入后，启动定时器：
    
    - (void)registerCountDown: (LXDTimerCallback)countDown
                   forSeconds: (NSUInteger)seconds
                 withReceiver: (id)receiver {
        if (countDown == nil || seconds <= 0 || receiver == nil) { return; }
        
        lxd_wait(self.lock);
        self.receives->insertReceiver((__bridge void *)receiver, countDown, seconds);
        [self _startupTimer];
        lxd_signal(self.lock);
    }
    
    bool LXDReceiverHashmap::insertReceiver(void *obj, LXDReceiverCallback callback, unsigned long lefttime) {
        unsigned int offset = obj_hash_code(obj);
        hash_entry_t *entry = hash_entries + offset;
        LXDReceiverNode *header = (LXDReceiverNode *)entry->entry;
        LXDReceiverNode *node = header->next;
        
        if (node == NULL) {
            LXDReceiver *receiver = create_receiver(obj, callback, lefttime);
            node = new LXDReceiverNode(receiver, header);
            header->next = node;
            header->count++;
            return true;
        }
        
        do {
            if (compare(node, obj) == true) {
                node->receiver->callback = callback;
                node->receiver->lefttime = lefttime;
                return false;
            }
        } while (node->next != NULL && (node = node->next));
        
        if (compare(node, obj) == true) {
            node->receiver->callback = callback;
            node->receiver->lefttime = lefttime;
            return false;
        }
        
        LXDReceiver *receiver = create_receiver(obj, callback, lefttime);
        node->next = new LXDReceiverNode(receiver, node);
        header->count++;
        return true;
    }
    
#### 倒计时回调
![](http://p0zs066q3.bkt.clouddn.com/2018010614.png)

定时器启动后，会遍历所有的回调链表，并且调起回调处理。如果在本次遍历中发生已经不存在任何倒计时任务，那么定时器将被释放：

    - (void)_countDown {
        NSMutableArray *removeNodes = @[].mutableCopy;
        
        for (unsigned int offset = 0; offset < _receives->entries_count; offset++) {
            hash_entry_t *entry = _receives->hash_entries + offset;
            LXDReceiverNode *header = (LXDReceiverNode *)entry->entry;
            LXDReceiverNode *node = header->next;
            
            while (node != NULL) {
                [removeNodes addObject: [NSValue valueWithPointer: (void *)node]];
                node = node->next;
            }
        }
        
        if (removeNodes == 0 && self.timer != nil) {
            lxd_wait(self.lock);
            dispatch_cancel(self.timer);
            self.timer = nil;
            lxd_signal(self.lock);
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            [removeNodes enumerateObjectsWithOptions: NSEnumerationReverse usingBlock: ^(NSValue *obj, NSUInteger idx, BOOL * _Nonnull stop) {
                LXDReceiverNode *node = (LXDReceiverNode *)[obj pointerValue];
                node->receiver->lefttime--;
                BOOL isStop = NO;
                
                node->receiver->callback(node->receiver->lefttime, &isStop);
                if (node->receiver->lefttime > 0 && !isStop) {
                    [removeNodes removeObjectAtIndex: idx];
                }
            }];
            
            dispatch_async(_timerQueue, ^{
                lxd_wait(self.lock);
                for (id obj in removeNodes) {
                    LXDReceiverNode *node = (LXDReceiverNode *)[obj pointerValue];
                    _receives->destoryNode(node);
                }
                lxd_signal(self.lock);
            });
        });
    }
    
倒计时任务会因为`倒计时归零`或者`标记位被修改`这两个原因结束，假如本次回调中正好所有的倒计时任务都处理完毕了，所有的注册者都被清除。此时并不会立刻停止定时器，而是等待到下次回调再停止。主要出于两个条件考虑：

1. 回调属于异步执行，如果要本次处理完成后检测注册队列状态，需要额外的同步机制开销
2. 假如在下次回调前又注册了新的倒计时任务，可以避免销毁重建定时器的开销

#### 前后台切换
应用在前后台切换的过程中，会在`非后台线程`执行完当前任务后挂起线程。一般来说，我们的倒计时会因为前后台切换而中止，除非我们`将倒计时放在主线程`或`创建后台线程继续执行`。此外，应用重新回到`ative`状态时，只要在后台停留的时长超出了定时器的回调间隔，那么倒计时会立刻被回调，破坏了原有的回调时间和倒计时长

![](https://user-gold-cdn.xitu.io/2018/1/6/160cbfcd58ef3116?imageView2/0/w/1280/h/960/ignore-error/1)

文章开头提到有三种方案解决这种前后台切换对定时器的方案。`后台线程倒计时`可以最大程度的保证倒计时的回调时间依旧正确，但是基于应用后台无感知的特性，这种消耗资源的方案不在我们的考虑范围。由于在设计上，我已经采用了保留`lefttime`的方式，因此`保存deadline`重新计算剩余时长也不是最佳选择。采用方案`2`计算后台停留时间并且更新剩余时间是最合适的做法：

    - (void)applicationDidBecameActive: (NSNotification *)notif {
        if (self.enterBackgroundTime && self.timer) {
            long delay = [[NSDate date] timeIntervalSinceDate: self.enterBackgroundTime];
            
            dispatch_suspend(self.timer);
            for (unsigned int offset = 0; offset < _receives->entries_count; offset++) {
                hash_entry_t *entry = _receives->hash_entries + offset;
                LXDReceiverNode *header = (LXDReceiverNode *)entry->entry;
                LXDReceiverNode *node = header->next;
                
                while (node != NULL) {
                    if (node->receiver->lefttime < delay) {
                        node->receiver->lefttime = 0;
                    } else {
                        node->receiver->lefttime -= delay;
                    }
                    
                    bool isStop = false;
                    node->receiver->callback(node->receiver->lefttime, &isStop);
                    if (node->receiver->lefttime <= 0 || isStop) {
                        lxd_wait(self.lock);
                        _receives->destoryNode(node);
                        lxd_signal(self.lock);
                    }
                }
            }
            dispatch_resume(self.timer);
        }
    }
    
    - (void)applicationDidEnterBackground: (NSNotification *)notif {
        self.enterBackgroundTime = [NSDate date];
    }

由于通知的回调线程和定时器的处理线程可能存在多线程的竞争，为了排除这一干扰，我采用了`sema`加锁，以及在遍历期间挂起定时器，减少不必要的麻烦

#### 对外接口
![](http://p0zs066q3.bkt.clouddn.com/2018010615.png)

前文说过，在不影响功能性的情况下，应当尽量减少对外接口的数量。因此，倒计时管理类只需要提供一个接口即可：

    /*!
     *  @class  LXDTimerManager
     *  定时器管理
     */
    @interface LXDTimerManager : NSObject
    
    /*!
     *  @method timerManager
     *  获取定时器管理对象
     */
    + (instancetype)timerManager;
    
    /*!
     *  @method registerCountDown:forSeconds:withReceiver:
     *  注册倒计时回调
     *
     *  @params countDown   回调block
     *  @params seconds     倒计时长
     *  @params receiver    注册的对象
     */
    - (void)registerCountDown: (LXDTimerCallback)countDown
                   forSeconds: (NSUInteger)seconds
                 withReceiver: (id)receiver;
    
    @end

#### 操作安全
为了倒计时任务的可靠性，我们应该在子线程启动定时器，一方面提高了精准度，另一方面避免造成主线程的卡顿。但由于涉及到`UI更新`和`前后台切换`两个情况，必须要考虑到多线程可能对数据的破坏力。从设计上来说，底层设计只提供实现接口，不考虑任何业务场景。因此应该在上层调用处做安全处理

![](http://p0zs066q3.bkt.clouddn.com/2018010606.png)

管理类使用`DISPATCH_QUEUE_SERIAL`属性创建的任务队列，确保定时器的回调之间是互不干扰的。对外提供的`register`接口无法保证调用方所处的线程环境，因此应当对操作进行加锁。此外涉及到`hashmap`的改动的代码都应当加锁保护：

    - (instancetype)init {
        if (self = [super init]) {
            self.receives = new LXDReceiverHashmap();
            self.lock = dispatch_semaphore_create(1);
            self.timerQueue = dispatch_queue_create("com.sindrilin.timer.queue", DISPATCH_QUEUE_SERIAL);
            
            [[NSNotificationCenter defaultCenter] addObserver: self selector: @selector(applicationDidBecameActive:) name: UIApplicationDidBecomeActiveNotification object: nil];
            [[NSNotificationCenter defaultCenter] addObserver: self selector: @selector(applicationDidEnterBackground:) name: UIApplicationDidEnterBackgroundNotification object: nil];
        }
        return self;
    }

    - (void)registerCountDown: (LXDTimerCallback)countDown
               forSeconds: (NSUInteger)seconds
             withReceiver: (id)receiver {
        if (countDown == nil || seconds <= 0 || receiver == nil) { return; }
        
        lxd_wait(self.lock);
        self.receives->insertReceiver((__bridge void *)receiver, countDown, seconds);
        [self _startupTimer];
        lxd_signal(self.lock);
    }
    
除了加锁，避免线程竞争的产生环境也是可行的。一个明显的竞争时机在于`应用切换前后台`和`倒计时回调`可能会同时被执行，因此在通知回调的遍历操作过程前后，将定时器`suspend`，避免恰好发生冲突的可能。

#### 循环引用
不同于大多数的倒计时方案，本方案通过扩充`NSObject`的方法来保证所有的类对象都可以注册倒计时任务。在`iOS`中，`block`是最容易引起循环引用的机制之一。为了尽量减少可能存在的引用问题，在接口的设计上，我让`block`接受一个`id`类型的调用对象，在接口层内部进行了一次`__weak`声明，并且在对象被释放后中止定时器任务：

    @implementation NSObject (PerformTimer)
    
    - (void)beginCountDown: (LXDObjectCountDown)countDown forSeconds: (NSInteger)seconds {
        if (countDown == nil || seconds <= 0) { return; }
        
        __weak typeof(self) weakself = self;
        [[LXDTimerManager timerManager] registerCountDown: ^(long leftTime, bool *isStop) {
            countDown(weakself, leftTime, (BOOL *)isStop);
        } forSeconds: seconds withReceiver: self];
    }

    @end
    
当然，我也做好了调用者完全不用`receiver`的准备了~

### 其他
最后再次声明这个观点：

> 倒计时方案几乎没有门槛，但也不仅限于倒计时方案

设计一个功能需要经过仔细考虑多个因素，包括`逻辑`、`性能`、`质量`多个方面。洋洋洒洒写完之后，发现设计倒计时也不是那么的容易，而且`hash + linked list`的设计上我采用了`struct + C++`的数据结构实现。虽然这套设计直接采用`NSDictionary + NSArray`来实现也是完全没有问题的，但是看了那么多源码，那么多算法，不去实践下实在太可惜了。基于`吐槽 + 实践`两层原因，最终完成了这么一个东西

***本篇文章基于这段时间学习的收获总结而成，如果您觉得有不足之处，还万请指出。项目已同步至`cocoapods`，可通过`pod 'LXDTimerManager'`导入***

> [demo](https://github.com/sindrilin/LXDTimerManager)

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)


