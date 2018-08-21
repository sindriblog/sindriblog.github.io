---
title: 线程安全
categories:
- Note
tags:
- thread
- lock
---

多线程技术对于计算机开发带来了巨大的性能提升，同样也来带了新的伤痛——线程安全问题。在开发中，稍不注意，我们就可能写出让代码陷入不安全的境地，线程锁是一种用来帮助我们保护临界资源的手段。事实上，现代语言提供了多种不同的线程锁来保护代码。通过深入挖掘，可以发现线程锁的核心无非是`Compare and Set`，基于这简单的核心，衍化出了多种安全方案，本文就来讲讲锁的原理


## 数据破坏
在理解如何保障临界代码的安全之前，我们需要了解数据为什么在多线程环境下被破坏。以简单的`i++`为例，这句代码将`i`自增一次，在编译成汇编代码后实际上会有三步操作：

    movl -0x24(%rbp), %r8d
    addl $0x1, %r8d
    movl %r8d, -0x24(%rbp)
    
完成一次`i++`总共分为三步（把大象放进冰箱）：

- 取出`i`存放到临时寄存器上

- 对寄存器的值`+1`

- 将计算后的值存放回`i`的内存

![](http://upload-images.jianshu.io/upload_images/783864-a75665306cb6fcad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

假设线程`A`执行`i++`这句代码，在完成将计算后的数值存储回`i`的内存之前，线程`B`也开始执行这句代码，最终的结果是两次`i++`之后，值仍然为`1`，这时候数据就发生了破坏。如果这时候使用线程锁，那么`B`就会等待`A`完成存储数据后才能执行：

![](http://upload-images.jianshu.io/upload_images/783864-ea32b1081852e76c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

加锁后，`B`在执行`i++`之前会检测指令是否被锁住。如果被锁住，则开始休眠，直到`A`完成操作后被唤醒继续执行代码。这时候`i++`在多线程环境下是安全的

## 原子性

> 所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch

我用`原子性`来表示某段多线程环境下的安全代码。从上面的代码来说，加锁之后的`i++`是具有`原子性`，因为代码在执行的过程中是线程安全的。此外，单条汇编一般也可以认为是具有`原子性`。而具有`原子性`的汇编指令也可以称作`原子操作`

> 原子（atom）指化学反应不可再分的基本微粒，原子在化学反应中不可分割

正常来说，代码分割到最小的单位就是单句汇编指令，比如上面的`addl $0x1 %r8d`可以被当做是代码中的`原子`。之所以说单条汇编是`原子操作`是因为在多线程环境下，汇编是不可再分割的，所以不会出现上面的破坏执行次序的问题。但是这并不是绝对的，比如系统中断可以中止正在执行的命令，这时候`%r8d`仍然是可能被意外修改。在汇编语言层面上，提供了`LOCK`指令前缀来保护指令执行过程层中的数据安全：

    lock addl $0x1 %r8d

除此之外，在`80486`指令集中还有`xadd`、`cmpxchg`和`xchg`等指令是多处理器安全的。加了`lock`修饰的单条编译指令以及这些特殊的安全指令才算是真正的`原子操作`

## 线程锁
单条的汇编指令可以通过`lock`来保证`原子操作`，有通过锁住地址总线的方式保证指令执行过程中的读取安全的手段。当然，鉴于笔者水平，也不多做深究。

而在非汇编指令的代码层面上来说，我们使用`互斥锁`、`自旋锁`、`条件锁`等等工具来保护代码安全。那么这些锁具体是怎么实现的呢？锁分为`信号量`和`互斥锁`，他们两者的使用区别如下：

- 互斥锁

    互斥锁应当是排它的，意思是锁在被某个线程获取之后，只有获取锁的线程才能释放这个锁。其他线程必须等到获取锁的线程不再拥有锁之后，才能继续执行。在我使用`NSLock`的测试中，发现可以`unlock`其他线程的锁，因此严格来说`NSLock`并不适合被称作互斥锁

- 信号量

    信号量拥有比互斥锁更多的用途。当信号量的`value`大于0时，所有的线程都能访问临界资源。在线程进入临界区后，`value`减一，反之亦然。如果信号量初始化为`0`时，可以看做是等待任务执行完成而非资源保护。`value`的操作应当是采用`原子操作`来保证指令的安全性的

## 互斥
锁的实现方式之一是互斥方式实现的（想了半天，还是决定用这个词）。即当线程`B`访问已经加锁了的临界资源时，检测到代码加锁，于是切换至内核态进行进一步的操作。伪代码大致实现如下，假定下面的代码是线程安全的：

    if (!lock.try_lock()) {
        /// 切换至内核态
        thread current = this;
        list queue = get_global_wait_list();
        queue.push(current);
        current.sleep(forever);
    }
    
此时，线程会进行休眠状态避免继续占用`CPU`资源，然后等待锁持有者执行完成释放锁。一旦任务完成，会检测是否存在等待执行代码的线程，如果存在，唤醒继续执行任务：

    list queue = get_global_wait_list();
    if ((t = queue.pop())) {
        t.wakeup();
    }
    
在具体实现中互斥的实现要复杂的多，但是不妨碍它基于一个简单的机制实现。互斥的实现涉及到了可能发生的内核态切换，线程休眠、唤醒等，如果临界执行代码足够小而快，互斥的线程锁可能并不是最佳的实践方案

## 自旋
自旋的实现要比互斥简单的多。对于自旋实现的线程锁来说，存在一个线程间共享的标记变量。当某个线程进入临界区后，变量被标记，此时其他线程再想进入临界区，会进入`while`循环中空转等待：

    while(flag) {
        continue;
    }
    
自旋的实现逻辑足够简单，只要标记位的修改被设计为`原子操作`，就能保证多线程环境下的安全。对比互斥方案，自旋没有线程切换、休眠唤醒的开销。但是空转的代码会导致`CPU`在等待期间是满负荷执行的，如果加锁的代码不够小而快，甚至会直接影响到程序的运行

## 信号
信号的性能在自旋和互斥之间，通常的性能表现总是仅次于自旋。这里基于`GCD`的信号量实现来看，在进入等待时，会根据传入的超时时间出现三种表现：

- `DISPATCH_TIME_NOW`

        while ((orig = dsema->dsema_value) < 0) {
            if (dispatch_atomic_cmpxchg2o(dsema, dsema_value, orig, orig + 1)) {
        #if USE_MACH_SEM
                return KERN_OPERATION_TIMED_OUT;
                
        #elif USE_POSIX_SEM || USE_FUTEX_SEM
                errno = ETIMEDOUT;
                return -1;
        #endif
            }
        }
        
- `DISPATCH_TIME_FOREVER`

        #if USE_MACH_SEM
        do {
            kr = semaphore_wait(dsema->dsema_port);
        } while (kr == KERN_ABORTED);
		DISPATCH_SEMAPHORE_VERIFY_KR(kr);
		
        #elif USE_POSIX_SEM
        do {
            ret = sem_wait(&dsema->dsema_sem);
        } while (ret == -1 && errno == EINTR);
        DISPATCH_SEMAPHORE_VERIFY_RET(ret);
        
        #elif USE_FUTEX_SEM
        do {
            ret = _dispatch_futex_wait(&dsema->dsema_futex, NULL);
        } while (ret == -1 && errno == EINTR);
        DISPATCH_SEMAPHORE_VERIFY_RET(ret);
        #endif

根据超时时间的设置，信号量最终会表现为`互斥`或者`自旋`的方式实现，这也是为什么评测中信号量性能总是优于`互斥`低于`自旋`。虽然信号量的性能不是最优，但是这种结合方案保证了它的作用范围更大

## barrier
`barrier`的任务总是保证在执行过程中，并发队列中有且只有`barrier`的任务在执行。最初笔者一度认为`barrier`的操作不过是加锁实现，后来在`libdispatch`的源码中得以窥见真容：

    void dispatch_barrier_async_f(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func)
    {
        dispatch_continuation_t dc;

        dc = fastpath(_dispatch_continuation_alloc_cacheonly());
        if (!dc) {
            return _dispatch_barrier_async_f_slow(dq, ctxt, func);
            }

        dc->do_vtable = (void *)(DISPATCH_OBJ_ASYNC_BIT | DISPATCH_OBJ_BARRIER_BIT);
        dc->dc_func = func;
        dc->dc_ctxt = ctxt;

        _dispatch_queue_push(dq, dc);
    }

相比`dispatch_async`的实现，`barrier`只是简单的将任务标记为`DISPATCH_OBJ_ASYNC_BIT`。但在执行队列任务的`_dispatch_queue_drain`会循环获取任务并且判断，`barrier`任务的真正实现在这个函数中。由于函数实现稍长，笔者只放上去除额外参数的伪代码：

    void _dispatch_queue_drain() {
        while((task = queue.next())) {
            if (queue.excute_barrier()) {
                return;
            } else if (task.do_vtable & DISPATCH_OBJ_ASYNC_BIT) {
              return;
            } else {
               task.execute();
            }
        }
    }

当循环取出队列任务执行的时候，检测到当前存在`barrier`的任务，则停止任务获取，直到当前所有的任务执行完成。并且在`barrier`执行过程中，不允许执行其他任务

## Compare and Set
上面总结了很多种线程锁方案，包括从伪代码和源代码窥探实现，线程锁的实现机制其实基于很简单的概念：`标志是否被占用`。而在这其中，核心确实无非`Compare and Set`，这两个是最核心的操作，通过`原子操作`实现这两个步骤来保证多线程锁的获取中不会出现另外的线程安全问题。笔者用八个字总结了线程锁的特性：

> 因为简单，所以可靠


## 最后
从入职新东家以来，深感到`经济基础决定上层建筑`这句话的意义。基础薄弱影响了笔者难以突破很多技术上的关口，是以未来很长一段时间都要将自己曾经欠下的债慢慢补上。同时，规范化的开发流程也是对自己的一个巨大挑战。我想，仅仅是实现功能就能自称工程师的话，无疑显得廉价。如何构建更稳健的代码，学会编写应对异常环境下的代码，等到那时候，我才有资格自称工程师

## 参考
[互斥锁的实现](http://blog.csdn.net/hzhzh007/article/details/6532988)

[用汇编实现原子操作](http://blog.csdn.net/zacklin/article/details/7445421)


![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)

