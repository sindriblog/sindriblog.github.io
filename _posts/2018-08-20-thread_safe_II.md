---
title: 线程安全（二）
date: 2018-08-20 08:00:00
tags:
- 多线程
---

之前写过一篇[线程安全](http://sindrilin.com/note/2017/09/09/thread_safe.html)，简单介绍了保护数据安全的多种方式，以及其中一部分方式的原理。基于此基础，本文将介绍如何避免锁的性能浪费，以及如何实现无锁安全结构

## 避免锁的性能浪费
为了避免多个线程对数据的破坏，在使用锁保障线程安全的情况下，存在几个影响锁性能的重要因素：

- 上下文切换
- 临界区资源耗时

如果能够减少这些因素的损耗，就能有效的提高锁的性能

### 自旋锁
通常来说，当一个线程获取锁失败后，会被添加到一个`等待队列`的末尾，然后休眠。直到锁被释放后，依次唤醒访问临界资源。休眠时会发生线程的上下文切换，当前线程的寄存器信息会被保存到磁盘上，考虑到这些情况，能做的有两点：

1. 换一个更快的磁盘
2. 改用`自旋锁`

`自旋锁`采用死循环等待锁释放来替代线程的休眠和唤醒，避免了上下文切换的代价。当临界的代码足够短，使用`自旋锁`对于性能的提升是立竿见影的

### 锁粒度
> 粒度是指颗粒的大小

对于锁来说，锁的粒度大小取决于锁保护的临界区的大小。锁的粒度越小，临界区的操作就越小，反之亦然，由于临界区执行代码时间导致的损耗问题我称作`粒度锁问题`。举个例子，假如某个修改元素的方法包括三个操作：`查找缓存`->`查找容器`->`修改元素`：
    
    - (void)modifyName: (NSString *)name withId: (id)usrId {
        lock();
        User *usr = [self.caches findUsr: usrId];
        if (!usr) {
            usr = [self.collection findUsr: usrId];
        }
        if (!usr) {
            unlock();
            return;
        }
        usr.name = name;
        unlock();
    }
    
实际上整个修改操作之中，只有最后的`修改元素`存在安全问题需要加锁，并且如果加锁的临界区代码执行时间过长可能导致有更多的线程需要等待锁，增加了锁使用的损耗。因此加锁代码应当尽量的短小简单：

    - (void)modifyName: (NSString *)name withId: (id)usrId {
        User *usr = [self.caches findUsr: usrId];
        if (!usr) {
            usr = [self.collection findUsr: usrId];
        }
        if (!usr) {
            return;
        }
        
        lock();
        usr.name = name;
        unlock();
    }

将`大段代码`改为`小段代码`加锁是一种常见的减少锁性能损耗的做法，因此不再多提。但接下来要说的是另一种常见但因为锁粒度造成损耗的问题：设想一下这个场景，在改良后的代码使用中，`线程A`对第三个元素进行修改，`线程B`对第四个元素进行修改：

![](https://upload-images.jianshu.io/upload_images/783864-6b02471c8091d582.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在两个线程修改`user`的过程中，实际上双方的操作是不冲突，但是`线程B`必须等待`A`完成修改工作，造成这个现象的原因是虽然看起来是对`usr.name`进行了加锁，但实际上是锁住了`collection`或`caches`的操作，所以避免这种隐藏的粒度锁问题的方案是以容器元素单位构建锁：包括`全局锁`和`独立锁`两种：

- `全局锁`

    构建一个`global lock`的集合，用`hash`的手段为修改元素对应一个锁：
    
        id l = [SLGlobalLocks getLock: hash(usr)];
        lock(l);
        usr.name = name;
        unlock(l);
        
    使用`全局锁`的好处包括可以在设计上可以懒加载生成锁，限制`bucketCount`来避免创建过多的锁实例，基于`hash`的映射关系，锁可以被多个对象获取，提高复用率。但缺点也是明显的，匹配锁的额外损耗，`hash`映射可能导致多个锁围观一个锁工作等。事实上`@synchronized`就是已存在的`全局锁`方案
    
- `独立锁`
    
    这个方案的名字是随便起的，从设计上要求容器的每个元素拥有自己的独立锁：
    
        @interface SLLockItem: NSObject
        
        @property (nonatomic, strong) id item;
        @property (nonatomic, strong) NSLock *lock;
        
        @end
        
        SLLockItem *item = [self.collection findUser: usrId];
        [item.lock lock];
        User *usr = item.item;
        usr.name = name;
        [item.lock unlock];

    `独立锁`保证了不同元素之间的加锁是绝对独立的，粒度完全可控，但锁难以复用，容器越长，需要创建的锁实例就越多也是致命的缺点。并且在链式结构中，`增删`操作的加锁是要和`previous`节点的修改操作发生竞争的，在实现上更加麻烦
     
## 无锁安全结构
`无锁化`是完全抛弃加锁工具，实现多线程访问安全的方案。`无锁化`需要去分解操作流程，找出真正需要保证安全的操作，举个例子：存在链表`A -> B -> C`，删除`B`的代码如下：

    Node *cur = list;
    while (cur.next != B && cur.next != nil) {
        cur = cur.next;
    }
    
    if (cur.next == nil) {
        return;
    }
    
    cur.next = cur.next.next;

![](https://upload-images.jianshu.io/upload_images/783864-cc6079ba9e8a378b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只要`A.next`的修改是不受多线程干扰的，那么就能保证删除元素的安全

### CAS
`compare and swap`是计算机硬件提供的一种原子操作，它会比较两个值是否相等，然后决定下一步的执行指令，`iOS`对于这种操作的支持需要导入`<libkern/OSAtomic.h>`文件。

    bool	OSAtomicCompareAndSwapPtrBarrier( void *oldVal, void *newVal, void * volatile *theVal )
    
函数会在`oldVal`和`theVal`相同的情况下将`oldVal`存储的值修改为`newVal`，因此删除`B`的代码只要保证在`A.next`变成`nil`之前是一致的就可以保证安全：

    Node *cur = list;
    while (cur.next != B && cur.next != nil) {
        cur = cur.next;
    }
    
    if (cur.next == nil) {
        return;
    }
    
    while (true) {
        void *oldValue = cur.next;
        if (OSAtomicCompareAndSwapPtrBarrier(oldValue, nil, &cur.next)) {
            break;
        } else {
            continue;
        } 
    }

基于上面删除`B`的例子，同一时间存在其他线程在`A`节点后追加`D`节点：

![](https://upload-images.jianshu.io/upload_images/783864-c425798ee2fdf856.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于`CPU`可能在任务执行过程中切换线程，如果`D`节点的修改工作正好在删除任务的中间完成，最终可能导致的是`D`节点的误删：

![](https://upload-images.jianshu.io/upload_images/783864-1acf37d2ef881b81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以上面的`CAS`还需要考虑`A.next`是否发生了改变：

    Node *cur = list;
    while (cur.next.value != B && cur.next != nil) {
        cur = cur.next;
    }
    
    if (cur.next == nil) {
        return;
    }
    
    while (true) {
        void *oldValue = cur.next;
        
        // next已经不再指向B
        if (!OSAtomicCompareAndSwapPtrBarrier(B, B, &cur.next.value)) {
            break;
        }
        
        if (OSAtomicCompareAndSwapPtrBarrier(oldValue, nil, &cur.next)) {
            break;
        } else {
            continue;
        } 
    }

## 题外话
`OSAtomicCompareAndSwapPtrBarrier`除了保证修改操作的原子性，还带有`barrier`的作用。在现在`CPU`的设计上，会考虑打乱代码的执行顺序来获取更快的执行速度，比如说：

    /// 线程1执行
    A.next = D;
    D.next = C;
    
    /// 线程2执行
    while (D.next != C) {}
    NSLog(@"%@", A.next);
    
由于执行顺序会被打乱，执行的时候变成：

    D.next = C;
    A.next = D;
    
    while (D.next != C) {}
    NSLog(@"%@", A.next);
    
输出的结果可能并不是`D`，而只要在`D.next = C`前面插入一句`barrier`函数，就能保证在这句代码前的指令不会被打乱执行，保证正确的代码顺序

## 最后
很方，这个月想了很多想写的内容，然后发现别人都写过，尴尬的一笔。果然还是自己太鶸了，最后随便赶工了一篇全是水货的文章，瑟瑟发抖

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)

