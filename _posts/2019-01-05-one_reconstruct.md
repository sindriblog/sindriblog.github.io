---
title: 记一次重构
date: 2019-01-05 08:00
tags:
- 开发笔记
---

## 技术重构
重构是软件开发过程中不断对软件代码进行打散重组，提高代码稳定性和可读性的处理手段之一。对【技术重构】进行关键信息提炼可以得到思维导图：

![](https://upload-images.jianshu.io/upload_images/783864-cd8ff8a95884d63c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

本文以微视最近一次音乐播放功能的重构为例回顾重构过程

## 重构步骤

### 业务梳理
微视`4.8`版本增加了音乐榜单功能，在更早之前的版本拥有音乐播放的界面只有音乐聚合页，相较于音乐聚合页同时只有一首歌曲需要控制播放，音乐榜单页存在切歌、榜单切换的场景，逻辑处置起来要棘手的多。另外由于音乐播放应该是一个通用能力，在重构前控制器需要维护`AVPlayer`的各种状态，代码格式如下：

    - (void)observeValueForKeyPath: (NSString *)keyPath 
                          ofObject: (id)object 
                            change: (NSDictionary<NSKeyValueChangeKey,id> *)change 
                           context: (void *)context {
        if ([UIApplication sharedApplication].applicationState != UIApplicationStateActive) {
            return;
        }
        
        if ([keyPath isEqualToString: @"rate"]) {
            NSNumber *val = change[NSKeyValueChangeNewKey];
            [self updateStateWhenRateChanged: val.doubleValue];
        }
        else if ([keyPath isEqualToString: @"status"]) {
            if (self.audioPlayer.currentItem.status != AVPlayerItemStatusReadyToPlay) {
                return;
            }
            [self updateStateWhenReadyToPlay];
        }
        else if ([keyPath isEqualToString: @"loadedTimeRanges"]) {
            NSArray *ranges = change[NSKeyValueChangeNewKey];
            [self updateStateWhenLoadedTimeChanged: ranges];
        }
    }
    
由于`KVO`存在强引用，为了避免存在的内存泄漏，还需要在控制器`disappear`的时候去移除监听：

    - (void)viewDidDisappear: (BOOL)animated {
        [super viewDidDisappear: animated];
        [self removeAudioPlayerObservers];
    }
    
    - (void)addAudioPlayerObservers {
        if (!self.audioPlayer) {
            return;
        }
        [self.audioPlayer addObserver: self forKeyPath: @"rate" options: NSKeyValueObservingOptionNew context: nil];
        [self.audioPlayer.currentItem addObserver: self forKeyPath: @"status" options: NSKeyValueObservingOptionNew context: nil];
        [self.audioPlayer.currentItem addObserver: self forKeyPath: @"loadedTimeRanges" options: NSKeyValueObservingOptionNew context: nil];
    }
    
    - (void)removeAudioPlayerObservers {
        if (!self.audioPlayer) {
            return;
        }
        self.audioPlayer removeObserver: self forKeyPath: @"rate" context: nil];
        self.audioPlayer.currentItem removeObserver: self forKeyPath: @"status" context: nil];
        self.audioPlayer.currentItem removeObserver: self forKeyPath: @"loadedTimeRanges" context: nil];
    }
    
对于音乐榜单页来说，由于音乐列表的存在，需要维护`loading`、`pause`、`playing`和`idle`四种状态值，以便能正确显示视图的状态。这种情况下将音乐播放分离出来有三个原因：

1. 多种改变播放状态的逻辑无法统一处理
2. 控制器管理了不属于自身的逻辑
3. 监听方式增加代码的维护成本

### 明确目标
由于`AVFoundation`提供的音乐播放器需要进行额外的配置以及维护状态，这部分的代码属于通用逻辑，不易放在`controller`这种业务上层中，因此通过添加一个间接层实现播放器的控制逻辑。计划重构后的结构如下：

                -------------------
    控制器       |  ViewController |
                -------------------
                         ↓
                         ↓
                -------------------
    请求层       |  WSMusicPlayer  |
                -------------------
                         ↓
                         ↓
                -------------------
    核心层       |   AVFoundation  |
                -------------------
                
分离后的播放器对外提供少量的控制接口，以及通过`delegate`统一状态变化的回调：

![](https://upload-images.jianshu.io/upload_images/783864-f620cf023ed8e8b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 动手实践
分离之后的`musicPlayer`主要有三处设计点：

> `KVO`的引用分离

`KVO`对象无法在`dealloc`中释放监听，因此在监听双方插入一个弱引用转发者，破坏循环引用链：
    
                        -------------
         -------------- |   Player  | ------------
         |               -------------           |
         ↓                                       ↓
    ------------                            -----------
    |   Proxy  |  ←-----------------------  |   Item  |
    ------------                            -----------

> 接口和实现的分离

控制接口包装了对实际操作的调用，提供过滤作用。通过`interface2`的方式命名实际接口，其存在的调用关系如下：

    - playWithURL:
        --> stop2
        --> play2
        --> switchOnOff
        
    - switchOnOff
        --> play2
        --> pause2
        
    - pause:
        --> pause2
        
    - stop:
        --> stop2

> 前后台切换的暂停续播

对外暴露`autoSwitchWhenApplicationStateChanged`配置音乐是否跟随前后台变化切换，默认跟随

## 踩的一些坑
1. 由于榜单页的多音乐播放场景会频繁的切换音乐，控制器虽然不再维护播放器的`state`，但依旧要维护当前播放的音乐。最开始`player`对外暴露`url`属性方便业务调用方判断，但发现这样给`player`开了一个口子，承担了不必要的向上层依赖风险，综合考虑之下由`play`接收参数进行转发
2. `notification`的`block`发生了引用。这是个低级错误，通知用多了，会下意识忘记了通知会存在引用



