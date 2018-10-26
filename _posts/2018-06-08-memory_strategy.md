---
title: WWDC-内存策略
date: 2018-06-08 08:00:00
tags:
- 开发笔记
---

尽管在进入后台之后，程序的工作受到大幅度的限制，但是我们总是不会希望应用突然被操作系统杀死，中断了重要的后台工作。后台应用被杀死，影响的不止是用户体验，比如正在播放的音乐戛然而止，正在导航的语音意外中断。由于操作系统杀死后台应用并不是任何一种异常、或者运行错误，应用失去响应意外的能力，极有可能破坏了单次运行中积累的重要数据。因此如何减少应用在后台运行被杀死，是一个值得思考的问题

## 后台应用如何被杀
抛开因为应用在后台停留太久被杀死之外，因为内存问题被杀死是最重要的原因。内存和其他硬件资源有很大的不同，区别在于：

> 当资源需求量多于资源所能供应的时，`CPU`表现为性能下降，而操作系统会为了回收内存杀死后台应用

我们可以把设备内存分成四块区域，包括`system`、`background`、`foreground`和`free`四部分。

- `system`在设备开机后是固定占用的，几乎不可能被我们的应用访问
- `background`属于被后台运行的应用使用，内存不足时会被回收
- `foreground`属于当前运行应用使用的内存
- `free`属于可分配内存，系统会优先分配这部分内存给应用使用

假设设备刚开机，用户直接打开你的应用，这时候设备的内存不包括`background`部分：

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e1d766148?w=1240&h=1031&f=jpeg&s=30359)

应用在这时退出前台，由于此时还有足够的`free`内存可分配，系统暂时不会收回内存：

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e1d81afc2?w=1240&h=1018&f=jpeg&s=51837)

然后用户打开了一个新的应用，但是新的应用需要大量内存，此时超出了设备的可用内存量：

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e1e69d910?w=1240&h=906&f=jpeg&s=44240)

于是系统向你的应用发出`memory warning`通知，系统回收了暂时不用的内存，然后有足够的内存给新应用使用：

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e1f173758?w=1240&h=1015&f=jpeg&s=50643)

又或者根本没发出`warning`，你的应用直接被杀死：

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e1f3f6a5c?w=1240&h=944&f=jpeg&s=47788)

## 内存警告
内存警告`memory warning`会在系统察觉到可能没有更多的`free`内存使用时发出，包括通知和代理两种方式的调起。通常来说，开发者和系统都会针对`warning`做应对措施，根据处理方的角色不同，我们可以将其分为`主动策略`和`被动策略`

### 内存分类
在了解两种应对策略的差异之前，我们需要了解对于每一个运行的应用来说，可被系统自动清除的内存分为两类：**（此处内存限制为主动分配，可随时释放的内存）**

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e5ba0b5e8?w=1240&h=1201&f=jpeg&s=83327)

- `Dirty`

    左侧正在展示的内容会被我们使用`array`或者`dictionary`存储起来。多数时候，为了更流畅的效果，我们可能会缓存页面外的数据，比如预加载单元格、预计算高度等。这些数据决定了当前应用的状态、内容等，是`活跃数据`。这部分内存不会被系统回收，会一直保存直到应用终止

- `Purgeable`

    右侧存储了可能使用过的数据，比如用户进入推荐功能后请求了大量的数据，使用`NSCache`缓存在内存中，以便重新进入推荐页时快速的展示内容，这部分数据是明显的`未使用数据`。这部分的内存会被系统直接清除后很快的重新分配使用

从内存的分类来看，减少`Dirty`数据的数量，增加`Purgeable`可以减少内存使用的压力

### 被动策略
之所以要先说被动策略，是因为操作系统在发出`warning`之前，已经做了一些工作来尝试解决问题，只有了解这些才能更好的帮助我们解决问题

- `NSPurgeableData`

    前面说了被标记为`purgeable`的内存会被系统直接清理数据然后重新分配使用，这个清理的过程发生在发出警告之前。如果在清理`purgeable`之后就有足够的内存可用了，就不会发出警告。`NSPurgeableData`是`NSData`的子类，它提供了类似`dispatch_group_t`的接口，使用一个整型变量存储使用状态，当值为`0`时，表示数据处于`未使用`状态

        NSPurgeableData *purgeableData = [[NSPurgeableData alloc] initWithBytes: fileData.bytes length: fileData.length];
        [purgeableData beginContentAccess];
        /// 值+1，开始使用数据
        
        ......
        
        [purgeableData endContentAccess];
        /// 值-1，结束使用数据
        
        ......
        /// 过了很长时间，如果无法访问数据，说明被回收，需要重新创建
        if (![purgeableData beginContentAccess]) {
            purgeableData = [[NSPurgeableData alloc] initWithBytes: fileData.bytes length: fileData.length];
        }
        [purgeableData beginContentAccess];

- `NSCache`

    `NSCache`提供了一套类似`dictionary`的简单接口让我们存取数据，同时允许我们设置一个缓存最大值。`NSCache`的数据被认为是`purgeable`类型，这些数据会在缓存达到最大值时以`LRU`算法淘汰使用最少者，也会被系统在发出警告前回收

- `memory warning`

    清理`purgeable`数据后，如果内存还是不够用，这时候系统就会向所有活跃应用发出内存警告，此时进入开发者的主动应对环节

### 主动策略
从操作上来说，`purgeable`的数据都需要开发者维护，更像是一种主动性的机制。但这个机制在内存不够用时，会被系统自动收回内存，而无需开发者进行更多额外的工作，因此被我归类到`被动策略`。主动策略更应该是接收到`memory warning`后，开发者进行的工作：

    - (void)applicationDidReceiveMemoryWarning:(UIApplication *)application {
        /// release data cannot keep the application running
        [self.database close];
    }
    
    - (void)didReceiveMemoryWarningNotification: (NSNotification *)notification {
        /// release data cannot keep the application running
        [self.dataSource removeAllObjects];
        [self.images removeAllObjects];
        [self cleanAllAnimatorViews];
    }

处理`warning`几乎是开发者的唯一有效手段，但是这种手段也并不完全可靠，如果依赖于系统发出的内存警告，存在几个风险点：

- `memory warning`是针对所有运行应用发出的，这意味着同一时间，`CPU`可能忙不过来处理你的工作
- 如果恰好属于运行应用队列的后列，在`purgeable`数据被回收、且其他应用响应完警告后，依旧没有足够内存，有可能直接被杀死而没有处理的机会

因此，即便处理警告几乎是唯一手段，依旧还得思考其他方式。假如我们从将一部分数据映射到磁盘上，应用可以以最小的内存占用运行

### 文件映射
文件映射不是万能药，每次修改数据都会同样更新到磁盘相同位置，即便`SSD`的存取能力已经很强大了，但依旧是不小的开销。你的数据应该总是满足这些要求，才考虑使用文件映射：

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e60dd7333?w=1240&h=485&f=jpeg&s=16620)

- 应用运行的必要数据。比如音乐应用的播放音乐信息、播放器类
- 构造或者还原的成本很高。如果成本不高，那么重新构建并没有什么损失
- 数据便于计算。如果数据总是变化的，会对文件大小的计算造成严重干扰
- 数据几乎不发生改变。变化越少，存储价值越高

这些条件决定了文件映射这种方式并不是适用于所有数据，基本适用于配置文件、图像数据、文本数据等，而这些数据也往往是占用内存的大头

![](https://user-gold-cdn.xitu.io/2018/6/8/163dff3e764ec8b8?w=1240&h=691&f=jpeg&s=18478)

另外如果数据量很大时，将数据整合成一个大数据段。同样数据大小的情况下，系统处理多块文件映射的内存效率要低的多。另外太多的虚拟内存块，也会导致系统强行杀死应用。`NSData`允许我们使用`options`提供文件映射的方式：

    typedef NS_OPTIONS(NSUInteger, NSDataReadingOptions) {
        NSDataReadingMappedIfSafe =   1UL << 0,
        NSDataReadingUncached = 1UL << 1,
        NSDataReadingMappedAlways API_AVAILABLE(macos(10.7), ios(5.0), watchos(2.0), tvos(9.0)) = 1UL << 3,
    };

## 其他
由于苹果的沙盒机制，我们过去总是更多的关注于应用内对于内存的使用情况，比较少关注应用在后台运行时可能遇到的内存问题，引用`WWDC`视频上苹果工程师的一句话：

> 你应该总是认为：在你应用之外的其他东西，会对你保持敌意并且有毁灭你的可能

因此，我们要学习如何合理的运用内存使用策略，来抵抗应用外的进攻。当然，最重要的是，我们可以理直气壮的告诉`QA`：哼，应用的内存使用降下来了！

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)

