---
layout: post
title: 探索scrollView的实现
categories: iOS
tags: iOS
author: SindriLin
---

* content
{:toc}

`UIScrollView`滚动视图，绝对算的上是iOS开发中最重要的控件，用来展示多于一个屏幕的内容，可以滚动显示超过屏幕外的内容的特性使其产生了更多强大的子类：`UITableView、UICollectionView、UITextView`等等。尽管功能如此强大，但是`scrollView`本质上只是一个`UIView`的黑魔法，本文将剖析`UIScrollView`这种强大特性的实现过程

图层渲染
----

这里不得不提到`UIView`和`CALayer`的关系。在`UIKit`框架中，`UIView`是所有界面元素的基础，我们页面上可见的控件几乎都是从这个类派生出来的。之所以说几乎意味着我们也可以不通过`UIView`及其子类的途径来展示一些页面效果，比如有渐变效果的进度条——通过`CALayer`直接完成。关于两者的具体区别以及关系，我们不在这里详说，只需要知道每一个`UIView`管理着一个`CALayer`，所有我们看到的内容都是由后者进行渲染的。

当我们添加子视图的时候，会基于当前视图的坐标系原点进行计算，然后在设置好的位置对子视图的`layer`进行渲染，假设现在添加一个`frame`为40, 40, 120, 40的按钮，那么渲染图示如下：
<span><img src="/images/探索SCROLLVIEW的实现/1.jpeg" width="800" height="500"></span>

在按钮添加到当前视图之前，按钮自身先进行了渲染，然后在距离父视图上边40点，左边40点的位置进行组合。正是因为这种组合的渲染模式，在顶层的视图总是会遮盖下层的视图。通过下面的视图组合流程，我们也能明白为什么创建的`view`的`bounds总是{0, 0, width, height}`

<span><img src="/images/探索SCROLLVIEW的实现/2.jpeg" width="800" height="500"></span>

根据上面的视图组合，我们试想一下，如果`button`的坐标是(100, 40)，那么这个按钮还会显示在`view`上面吗？答案是肯定的，因为根据上面视图组合的实现，我们可以得出一个结论：当前的视图也存在一个父视图，父视图也存在其所在的父视图。如此循环，直到这个视图是`keyWindow`为止。那么我们就有下面的结构图示
<span><img src="/images/探索SCROLLVIEW的实现/3.jpeg" width="800" height="500"></span>



因此按钮是处在我们可视的范围内的。但是，按照这种组合方式，`scrollView`的实现就显得非常的神奇了，因为在`scrollView`上面的子视图一旦超过了它的显示范围。这里需要说到`view`的`clipsToBounds`或`layer`的`maskToBounds`属性，这两个属性尽管名字不一样，但是如果你在堆栈调用的时候进行调试，会发现最终调用的是`maskToBounds`方法。这两个值任意一个设置`YES`的时候，在上面视图组合的③步骤中，超出父视图范围内的部分将不进行渲染。

那么`scrollView`是否跟我们猜测的一样，通过设置`maskToBounds`这个值来屏蔽超出其显示范围的子视图呢？如果是的话，那么`scrollView`就只是一个普通的`UIView`。我们通过下面的代码验证

	UIScrollView * scrollView = [[UIScrollView alloc] initWithFrame: CGRectMake(0, 0, 200, 180)];
	scrollView.backgroundColor = [UIColor orangeColor];
	scrollView.center = self.view.center;
	[self.view addSubview: scrollView];
	
	UIView * subview = [[UIView alloc] initWithFrame: CGRectMake(60, 60, 180, 180)];
	subview.backgroundColor = [UIColor blueColor];
	[scrollView addSubview: subview];

这时候scrollView的效果是这样的：

<span><img src="/images/探索SCROLLVIEW的实现/4.jpeg" width="800" height="500"></span>



接下来设置`scrollView`的`maskToBounds`属性

	scrollView.layer.masksToBounds = NO;

效果图

<span><img src="/images/探索SCROLLVIEW的实现/5.jpeg" width="800" height="500"></span>



可以看到，`scrollView`本质上不过是一个默认遮盖范围外子视图的`UIView`罢了。那么，`UIView`到底使用了什么黑魔法来实现滚动视图呢？

contentOffset
----
用过`scrollView`的开发者对这个属性都不陌生，`contentOffset`决定了当前`scrollView`显示内容的范围，即是当前`scrollView`的左上角的显示位置坐标。通过图片轮播控件来探究这个属性的实现

<span><img src="/images/探索SCROLLVIEW的实现/6.jpeg" width="800" height="500"></span>

上图中`scrollView`发生了滚动，使得显示的图片从1变成2。在这个过程中，`contentOffset`也从`(0, 0)`变为`(width, 0) `从这张图上看更像是子视图的位置发生了移动，从右向左移动。但是在这一切发生的过程中，子视图的`frame``没有发生过任何变化，因此与其说是滚动，不如说是`scrollView`基于子视图的所在的坐标系发生了偏移：

<span><img src="/images/探索SCROLLVIEW的实现/7.jpeg" width="800" height="500"></span>

两张图都表示了图片轮播的过程，但是第二张更加接近`scrollView`滚动实现的本质——基于自身的坐标系发生了位置偏移。因此，`contentOffset`实际上表示的是`scrollView的bounds`的改变，其实现大概如下

	- (void)setContentOffset: (CGPoint)contentOffset 
	{
		_contentOffset = contentOffset;
		CGRect bounds = self.bounds;
		bounds.origin = contentOffset;
		self.bounds = bounds;
	}
  
contentSize
----
  
  如果说`contentOffset`决定了`scrollView`的窗口，那么`contentSize`决定了这个窗口背后的风光。
  
  <span><img src="/images/探索SCROLLVIEW的实现/8.jpeg" width="800" height="500"></span>
  
  `contentSize`决定了`scrollView`显示内容的尺寸范围，从上图看，我们可以知道，在`contentSize`的宽度或者长度任意一个尺寸大于`scrollView`等边长度的时候，`scrollView`才能实现滚动效果。当然了，单单是`contentSize`是不足以让我们实现`scrollView`的滚动范围限制的，这是`contentSize`和`contentOffset`的共同实现效果：
  
  <span><img src="/images/探索SCROLLVIEW的实现/9.jpeg" width="800" height="500"></span>
  
  ​

contentInset
----
`contentInset`是一个相当有用的属性，我在做的一个毛玻璃效果导航栏上下拉效果时就通过这个属性实现。这个属性可以在某种意义上增加或者减少我们的滚动尺寸范围：

<span><img src="/images/探索SCROLLVIEW的实现/10.jpeg" width="800" height="500"></span>



可以看到，`contentInset`让我们原本`contentOffset`和`contentSize`协同作用的滚动范围发生了改变，原本最上角`(0, 0)`的限制坐标变成了`(-contentInset.left, -contentInset.top)`

既然`contentInset`只是简单的改变了滚动范围的规则，为什么我们不直接通过`contentSize`来实现呢？这是由于更多时间，我们还需要在滚动视图的某个方向上面留下一块空白的区域进行自定义，这时候直接设置`contentInset`是最快的方式。而换成`contentSize`来实现，我们还必须同时改变`bounds`跟`center`来实现（不要直接改变frame，在组合视图时，frame最后是由bounds和center决定的）

尾话

苹果对于`scrollView`的实现十分的巧妙，在没有造成过多损耗的情况下赋予`UIView`一份强大无比的力量。解剖`UIKit`不仅仅是为了探索实现，这对于我们自定义控件能有更多的认识。在`scrollView`更上一层的`UITableView`通过复用队列的方式将`scrollView`的能力更加完美的展示出来，而这个复用机制值得我们去思考实现过程。

demo链接：[实现scrollView](https://github.com/JustKeepRunning/LXDScrollView)
转载请注明本文地址及作者

