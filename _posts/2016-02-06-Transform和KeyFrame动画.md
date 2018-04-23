---
layout: post
title: Transform和KeyFrame动画
categories: 动画
tags: 动画
author: SindriLin
---

* content
{:toc}


追求美好是人的天性，这是猿们无法避免的。我们总是追求更为酷炫的实现，如果足够仔细，我们不难发现一个好的动画通过步骤分解后本质上不过是一个个简单的动画实现，正是这些基本的动画在经过合理的搭配组合后化腐朽为神奇，令人惊艳。因此，掌握最基本的动画是完成酷炫开发之旅的根本。

作为动画篇的第二篇文章，我在[从UIView动画说起](http://www.jianshu.com/p/6e326068edeb)简单介绍了关于UIView的几种基本动画，这几种动画的搭配让我们的登录界面富有灵性生动，但是这几种动画总是无法满足我们对于动画的需求。同样的，本文将从一个小demo开始讲解强大的`transform`动画以及关键帧`keyFrame`动画。]

<span><img src="/images/Transform和KeyFrame动画/1.gif" width="800"></span>



可以看到两个动画：叶子被风吹落以及左边的文字从`summer`变化到`autumn`，这两个动画都是基于强大的`transform`形变，其中叶子的飘落动画通过关键帧动画实现。[demo链接](https://github.com/JustKeepRunning/LXDAnimationSecondDemo)

 transform动画
----

`transform`是一个非常重要的属性，它在矩阵变换的层面上改变视图的显示效果，完成旋转、形变、平移等等操作。在它被修改的同时，视图的frame也会被真实改变。有两个数据类型用来表示`transform`，分别是`CGAffineTransform`和`CATransform3D`。前者作用于`UIView`，后者为`layer`层次的变换类型。基于后者可以实现更加强大的功能，但我们需要先掌握`CGAffineTransform`类型的使用。同时，本文讲解也是这个变换类型。

对于想要了解矩阵变换是如何作用实现的，可以参考这篇博客：[CGAffineTransform 放射变换](http://blog.csdn.net/dyllove98/article/details/9051139)

>  talk is cheap show you the code

在开始使用`transform`实现你的动画之前，我先介绍几个常用的函数：

``` 
/// 用来连接两个变换效果并返回。返回的t = t1 * t2
CGAffineTransformConcat(CGAffineTransform t1, CGAffineTransform t2)

/// 矩阵初始值。[ 1 0 0 1 0 0 ]
CGAffineTransformIdentity

/// 自定义矩阵变换，需要掌握矩阵变换的知识才知道怎么用。参照上面推荐的原理链接
CGAffineTransformMake(CGFloat a, CGFloat b, CGFloat c, CGFloat d, CGFloat tx, CGFloat ty)

/// 旋转视图。传入参数为 角度 * (M_PI / 180)。等同于 CGAffineTransformRotate(self.transform, angle)
CGAffineTransformMakeRotation(CGFloat angle)
CGAffineTransformRotate(CGAffineTransform t, CGFloat angle)

/// 缩放视图。等同于CGAffineTransformScale(self.transform, sx, sy)
CGAffineTransformMakeScale(CGFloat sx, CGFloat sy)
CGAffineTransformScale(CGAffineTransform t, CGFloat sx, CGFloat sy)

/// 缩放视图。等同于CGAffineTransformTranslate(self.transform, tx, ty)
CGAffineTransformMakeTranslation(CGFloat tx, CGFloat ty)
CGAffineTransformTranslate(CGAffineTransform t, CGFloat tx, CGFloat ty)
```

我把demo左下角文字的变形过程记录下来。这里推荐mac上面的一款截取动图的程序[licecap](http://www.pc6.com/mac/135257.html)，非常简单好用。博主用它来分解动画步骤，然后进行重现。

<span><img src="/images/Transform和KeyFrame动画/1.jpeg" width="800"></span>



不难看出在文字的动画中做了两个处理：y轴上的形变缩小、透明度的渐变过程。首先在项目中新增两个UILabel，分别命名为label1、label2.然后在viewDidAppear中加入这么一段代码：

``` 
- (void)viewDidAppear: (BOOL)animated {
    label1.transform = CGAffineTransformMakeScale(0, 0);
    label1.alpha = 0;
    [UIView animateWithDuration: 3. animations: ^ {
        label1.transform = CGAffineTransformMakeScale(0, 1);
        label2.transform = CGAffineTransformMakeScale(0, 0.1);
        label1.alpha = 1;
        label2.alpha = 0;
    }];
}
```

这里解释一下为什么label2为什么在动画中y轴逐渐缩小为0.1而不是0。如果我们设为0的话，那么在动画提交之后，label2会直接保持动画结束的状态（这是出于性能优化自动完成的），因此在使用任何缩小的形变时，你可以将缩小值设置的很小，只要不是0。

运行你的代码，文字的形变过程你已经做出来了，但是demo中的动画不仅仅是形变，还包括位移的过程。很显然，我们可以通过改变`center`的位置来实现这个效果，但这显然不是我们今天想要的结果，实现新的动画方式来实现更有意义。

动画开始时形变出现的label高度为0，然后逐渐的的变高变为`height`，而label从头到尾基于顶部的位置不发生改变。因此动画开始前这个label在y轴上的位置是0，在完成显示之后的y轴中心点为`height / 2`（基于label自身的坐标系而言），那么动画的代码就可以写成这样：

``` 
- (void)viewDidAppear: (BOOL)animated {
    ///  初始化动画开始前label的位置
    CGFloat offset = label1.frame.size.height * 0.5;

    label1.transform = CGAffineTransformConcat(
      CGAffineTransformMakeScale(0, 0),
      CGAffineTransformTranslate(0, -offset)
    );
    label1.alpha = 0;
    [UIView animateWithDuration: 3. animations: ^ {
        ///  还原label1的变换状态并形变和偏移label2
        label1.transform = CGAffineTransformIdentifier;
        label1.transform = CGAffineTransformConcat(
          CGAffineTransformMakeScale(0, 0),
          CGAffineTransformTranslate(0, offset)
        );
        label1.alpha = 1;
        label2.alpha = 0;
    }];
}
```

调整两个label的位置，并且设置其中一个透明显示。然后运行这段代码，你会发现文字转变过程的动画完成了。

 keyframe动画
----

将文章开头的gif图另存为到本地，然后使用预览打开看看，你会发现预览中的gif图变成了很多张的图片。实际上，无论是动画、电影、CG等动态效果，都可以看做是一张张图片接连渲染实现的，而这些图片切换的速度足够快时我们就会当做是动画。在此之前我们所讲述的平移视图在UIView动画提交之后系统会根据动画时长计算出视图移动的所有帧界面，然后逐个渲染。

回到我们demo中的落叶动画来，我总共对叶子的`center`进行过五次修改，我将落叶平移的线性路径绘制出来并且标注关键的转折点：

<span><img src="/images/Transform和KeyFrame动画/2.jpeg" width="800"></span>



上面这个平移用UIView动画代码要如何实现呢？毫无疑问，我们需要不断的嵌套UIView动画的使用来实现，具体代码如下：

``` 
[self moveLeafWithOffset: (CGPoint){ 15, 80 } completion: ^(BOOL finished) {
    [self moveLeafWithOffset: (CGPoint){ 30, 105 } completion: ^(BOOL finished) {
        [self moveLeafWithOffset: (CGPoint){ 40, 110 } completion: ^(BOOL finished) {
            [self moveLeafWithOffset: (CGPoint){ 90, 80 } completion: ^(BOOL finished) {
                [self moveLeafWithOffset: (CGPoint){ 80, 60 } completion: nil duration: 0.6];
            } duration: 1.2];
        } duration: 1.2];
    } duration: 0.6];
} duration: 0.4];

- (void)moveLeafWithOffset: (CGPoint)offset completion: (void(^)(BOOL finished))completion duration: (NSTimeInterval)duration
{
    [UIView animateWithDuration: duration delay: 0 options: UIViewAnimationOptionCurveLinear animations: ^{
        CGPoint center = _leaf.center;
        center.x += offset.x;
        center.y += offset.y;
        _leaf.center = center;
    } completion: completion];
}
```

看起来还蛮容易的，上面的代码只是移动叶子，在gif图中我们的叶子还有旋转，因此我们还需要加上这么一段代码：

``` 
[UIView animateWithDuration: 4 animations: ^{
    _leaf.transform = CGAffineTransformMakeRotation(M_PI);
}];
```

那么ok，运行这段代码看看，落叶的移动非常的生硬，我们可以明显的看到拐角。其次，这段代码中的`duration`传入是没有任何意义的（传入一个固定的动画时长无法体现出在落叶飘下这一过程中的层次步骤）

对于这两个问题，UIView也提供了另一种动画方式来帮助我们解决这两个问题 —— keyframe动画：

``` 
+ (void)animateKeyframesWithDuration:(NSTimeInterval)duration delay:(NSTimeInterval)delay options:(UIViewKeyframeAnimationOptions)options animations:(void (^)(void))animations completion:(void (^ __nullable)(BOOL finished))completion
+ (void)addKeyframeWithRelativeStartTime:(double)frameStartTime relativeDuration:(double)frameDuration animations:(void (^)(void))animations
```

第一个方法是创建一个关键帧动画，第二个方法用于在动画的代码块中插入关键帧动画信息，两个参数的意义表示如下：

- frameStartTime  表示关键帧动画开始的时刻在整个动画中的百分比
- frameDuration  表示这个关键帧动画占用整个动画时长的百分比。

我做了一张图片来表示参数含义：

<span><img src="/images/Transform和KeyFrame动画/3.jpeg" width="800"></span>



对比`UIView`动画跟关键帧动画，关键帧动画引入了动画占比时长的概念，这让我们能控制每个关键帧动画的占用比例而不是传入一个无意义的动画时长 —— 这让我们的代码更加难以理解。当然，除了动画占比之外，关键帧动画的`options`参数也让动画变得更加平滑，下面是关键帧特有的配置参数：

``` 
UIViewKeyframeAnimationOptionCalculationModeLinear      // 连续运算模式，线性
UIViewKeyframeAnimationOptionCalculationModeDiscrete    // 离散运算模式，只显示关键帧
UIViewKeyframeAnimationOptionCalculationModePaced       // 均匀执行运算模式，线性
UIViewKeyframeAnimationOptionCalculationModeCubic       // 平滑运算模式
UIViewKeyframeAnimationOptionCalculationModeCubicPaced  // 平滑均匀运算模式
```

在demo中我使用的是`UIViewKeyframeAnimationOptionCalculationModeCubic`，这个参数使用了贝塞尔曲线让落叶的下落动画变得更加平滑。效果可见最开始的gif动画，你可以修改demo传入的不同参数来查看效果。接下来我们就根据新的方法把上面的`UIView`动画转换成关键帧动画代码，具体代码如下：

``` 
[UIView animateKeyframesWithDuration: 4 delay: 0 options: UIViewKeyframeAnimationOptionCalculationModeLinear animations: ^{
    __block CGPoint center = _leaf.center;
    [UIView addKeyframeWithRelativeStartTime: 0 relativeDuration: 0.1 animations: ^{
        _leaf.center = (CGPoint){ center.x + 15, center.y + 80 };
    }];
    [UIView addKeyframeWithRelativeStartTime: 0.1 relativeDuration: 0.15 animations: ^{
        _leaf.center = (CGPoint){ center.x + 45, center.y + 185 };
    }];
    [UIView addKeyframeWithRelativeStartTime: 0.25 relativeDuration: 0.3 animations: ^{
        _leaf.center = (CGPoint){ center.x + 90, center.y + 295 };
    }];
    [UIView addKeyframeWithRelativeStartTime: 0.55 relativeDuration: 0.3 animations: ^{
        _leaf.center = (CGPoint){ center.x + 180, center.y + 375 };
    }];
    [UIView addKeyframeWithRelativeStartTime: 0.85 relativeDuration: 0.15 animations: ^{
        _leaf.center = (CGPoint){ center.x + 260, center.y + 435 };
    }];
    [UIView addKeyframeWithRelativeStartTime: 0 relativeDuration: 1 animations: ^{
        _leaf.transform = CGAffineTransformMakeRotation(M_PI);
    }];
} completion: nil];
```

可以看到相比`UIView`的动画，关键帧动画更加直观的让我们明白每一次平移动画的时间占比，代码也相对的更加简洁。

 尾言
----

本文作为动画篇的第二篇博客，到了这里`UIView`的所有动画教程已经完成，在之后的文章中将进一步讲解`autolayout`动画和图层层次的动画。时值新年，祝愿各位🐵年快乐，心想事成！[本文demo地址](https://github.com/JustKeepRunning/Animations)

*转载请注明地址和原文作者*

