---
title: 认识CoreAnimation
categories:
- 动画
tags:
- 动画
---

在iOS中，普通的动画可以使用`UIKit`提供的方法来实现动画，但如果想要实现复杂的动画效果，使用`CoreAnimation`框架提供的动画效果是最好的选择。那么两种动画方案相比之下，后者存在的主要好处包括不仅下面这些：

- 轻量级的数据结构，可以同时让上百个图层产生动画效果
- 拥有独立的线程用于执行我们的动画接口
- 完成动画配置后，核心动画会代替我们完全控制完成对应的动画帧
- 提高应用性能。只有在发生改变的时候才重绘内容，消除了动画的帧速率上的运行代码

<span><img src="/images/认识CoreAnimation/1.jpeg" width="800"></span>

在`CoreAnimation`框架下，最主要的两个部分是图层`CALayer`以及动画`CAAnimation`类。前者管理着一个可以被用来实现动画的位图上下文；后者是一个抽象的动画基类，它提供了对`CAMediaTiming`和`CAAction`协议的支持，方便子类实例直接作用于`CALayer`本身来实现动画效果。接下来笔者会分段分别讲述上面提到的类，参考信息来自于[苹果官方文档](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Instrument-CoreAnimation.html)以及[objc中国](http://objccn.io)

 CALayer
----

> CALayer类结构

如果你喜欢动画效果，在网上开源的动画实现中总是能看到`CALayer`及其子类的应用，那么了解这个图层类别先从它的结构看起（此处列出了了部分属性并且去除了注释）：

	public class CALayer : NSObject, NSCoding, CAMediaTiming {
	
	    public func presentationLayer() -> AnyObject?
	    public func modelLayer() -> AnyObject
	
	    public var bounds: CGRect
	    public var position: CGPoint
	    public var anchorPoint: CGPoint
	    public var transform: CATransform3D
	    public var frame: CGRect
	    public var hidden: Bool
	
	    public var superlayer: CALayer? { get }
	    public func removeFromSuperlayer()
	    public func addSublayer(layer: CALayer)
	    public func insertSublayer(layer: CALayer, below sibling: CALayer?)
	    public func insertSublayer(layer: CALayer, above sibling: CALayer?)
	    public func replaceSublayer(layer: CALayer, with layer2: CALayer)
	    public var sublayerTransform: CATransform3D
	
	    public var mask: CALayer?
	    public var masksToBounds: Bool
	
	    public func hitTest(p: CGPoint) -> CALayer?
	    public func containsPoint(p: CGPoint) -> Bool
	
	    public var shadowColor: CGColor?
	    public var shadowOpacity: Float
	    public var shadowOffset: CGSize
	    public var shadowRadius: CGFloat
	
	    public var contents: AnyObject?
	    public var contentsRect: CGRect
	
	    public var cornerRadius: CGFloat
	    public var borderWidth: CGFloat
	    public var borderColor: CGColor?
	    public var opacity: Float
	}

根据[CALayer Class Reference](https://developer.apple.com/library/tvos/documentation/GraphicsImaging/Reference/CALayer_class/)中的描述，在每一个`UIView`的背后都有一个`CALayer`对象用来协助它显示内容，它自身管理着我们提供给视图显示的位图上下文以及保存这些位图上下文的几何信息。通过上面的代码可以看出：

- `CALayer`是`NSObject`的子类而非`UIResponder`的子类，因此图层本身无法响应用户操作事件却拥有着[事件响应链](http://www.jianshu.com/p/a8926633837b)相似的判断方法，所以`CALayer`需要包装成一个`UIView`容器来完成这一功能。
- 每一个`UIView`自身存在一个`CALayer`来显示内容。在后者的属性中我们可以看到存在着多个和`UIView`界面属性对应的变量，因此我们在修改`UIView`的界面属性的时候其实是修改了这个`UIView`对应的`layer`的属性。
- `CALayer`拥有和`UIView`一样的树状层级关系，也有类似`UIView`添加子视图的`addSublayer`这些类似的方法。`CALayer`可以独立于`UIView`之外显示在屏幕上，但我们需要重写事件方法来完成对它的响应操作

对于苹果为什么要把`UIView`和`CALayer`区分开来，网上已经有了一篇很详(qi)细(pa)的文章讲解这个问题[都有了CALayer，为什么还要UIView](http://www.cocoachina.com/ios/20150828/13257.html)

> 图层树和隐式动画

在每一个`CALayer`中，都有三个重要的层次树，它们负责相互协调完成图层的渲染展示效果。这三个层次树分别是：

- 模型树。通过`layer.modelLayer`获取，当我们修改`CALayer`的可动画属性时，模型树对应的属性就会立刻被修改成对应的数值
- 呈现树。通过`layer. presentationLayer`获取，呈现树保存着当前图层状态的显示数据，即会随着动画的过程不断更新图层的状态数据
- 渲染树，iOS并没有提供任何API来获取这一个层次树。顾名思义，它通过结合 `modelLayer`跟`presentationLayer`中设置的效果来将内容渲染到屏幕上

`CALayer`中的显示数据几乎都是可动画属性，这个特性为我们制作核心动画提供了很大的实践基础。在一个单独的`CALayer`中（也就是说这个`layer`并没有和任何`UIView`绑定），我们修改它的显示属性的时候，都会触发一个从`旧值`到`新值`之间的简单动画效果，这种动画我们称之为隐式动画：

	class ViewController: UIViewController {
	
	    let layer = CAShapeLayer()
	
	    override func viewDidLoad() {
	        super.viewDidLoad()
	        layer.strokeEnd = 0
	        layer.lineWidth = 6
	        layer.fillColor = UIColor.clearColor().CGColor
	        layer.strokeColor = UIColor.redColor().CGColor
	        self.view.layer.addSublayer(layer)
	    }
	
	    @IBAction func actionToAnimate() {
	        layer.path = UIBezierPath(arcCenter: self.view.center, radius: 100, startAngle: 0, endAngle: 2*CGFloat(M_PI), clockwise: true).CGPath
	        layer.strokeEnd = 1
	    }
	}

可以看到上面的代码中我单独创建了一个`CALayer`并且将它添加到当前的控制器视图的图层上，`strokeEnd`这一属性表示填充百分比。当这个属性发生变化的时候，产生了一个画圈的动画效果：

<span><img src="/images/认识CoreAnimation/1.gif" width="800"></span>

在隐式动画的实现背后，隐藏着一个最重要的扮演角色`CAAction`协议这一过程会在下面进行详细的介绍。那么在上面这个隐式动画的过程中，模型树和呈现树发生了哪些变化呢？由于系统的动画时长默认为`0.25`秒，我设置一个`0.05`秒的定时器在每次回调的时候查看一下这两个层次树的信息：
 
	@IBAction func actionToAnimate() {
	    timer = NSTimer(timeInterval: 0.05, target: self, selector: selector(timerCallback), userInfo: nil, repeats: true)
	    NSRunLoop.currentRunLoop().addTimer(timer!, forMode: NSRunLoopCommonModes)
	    layer.path = UIBezierPath(arcCenter: self.view.center, radius: 100, startAngle: 0, endAngle: 2*CGFloat(M_PI), clockwise: true).CGPath
	    layer.strokeEnd = 1
	}
	
	@objc private func timerCallback() {
	    print("========================\nmodelLayer: \t\(layer.modelLayer().strokeEnd)\ntpresentationLayer: \t\(layer.presentationLayer()!.strokeEnd)")
	    if fabs((layer.presentationLayer()?.strokeEnd)! - 1) < 0.01 {
	        if let _ = timer {
	            timer?.invalidate()
	            timer = nil
	        }
	    }
	}

控制台的输出结果如下：

	========================
	modelLayer: 	1.0
	presentationLayer: 	0.294064253568649
	========================
	modelLayer: 	1.0
	presentationLayer: 	0.676515340805054
	========================
	modelLayer: 	1.0
	presentationLayer: 	0.883405208587646
	========================
	modelLayer: 	1.0
	presentationLayer: 	0.974191427230835
	========================
	modelLayer: 	1.0
	presentationLayer: 	0.999998211860657

可以看到当一个隐式动画发生的时候，`modelLayer`的属性被修改成动画最终的结果值。而系统会根据动画时长和最终效果值计算出动画中每一帧的数值，然后依次更新设置到`presentationLayer`当中。最终这些计算工作都完成之后，渲染树`renderingTree`根据这些值将动画效果渲染到屏幕上。

那么通过层次树我们能制作什么呢？假设我需要制作下面这么一个粘性的弹球动画，那么我在界面的最左侧、最右侧以及中间各自添加了一个`CALayer`，当点击按钮的时候给左右两侧的`layer`添加一个匀速的`position`下移动画，中间的`centerLayer`添加一个弹簧动画。通过使用定时器更新获取这三个`layer`的呈现树`y`轴坐标来绘制区域形成这样一个动画：

<span><img src="/images/认识CoreAnimation/2.gif" width="800"></span>

> 其他属性

除了上面重点介绍的属性之外，下面的属性我只进行简单的介绍，详细的使用以及动画作用会在以后对应使用的动画中更详细的讲解：

- `position`和`anchorPoint`
  
  `anchorPoint `是一个`x`和`y`值取值范围内在`0~1`之间`CGPoint`类型，它决定了当图层发生几何仿射变换时基于的坐标原点。默认情况下为`0.5, 0.5`，由`anchorPoint `和`frame`经过计算获得图层的`position`这个值。更多介绍这两个属性的文章在[彻底理解position和anchorPoint](http://www.tuicool.com/articles/MvI7fu3)
  
- `mask`和`maskToBounds`
  
  `maskToBounds`值为`true`时表示超出图层范围外的所有子图层都不会进行渲染，当我们设置`UIView`的`clipsToBounds`时实际上就是在修改`maskToBounds`这个属性。`mask`这个属性表示一个遮罩图层，在这个遮罩之外的内容不予渲染显示，在上一篇动画[碎片动画](http://www.jianshu.com/p/e189696dd535)中使用的`maskView`实际上也是修改这个属性
  
- `cornerRadius`、`borderWidth`和`borderColor`
  
    `borderWidth`和`borderColor`设置了图层的边缘线条的颜色以及宽度，正常情况下这两个属性在`layer`的层次上不怎么使用。后者`cornerRadius`设置圆角半径，这个半径会影响边缘线条的形状
  
- `shadowColor`、`shadowOpacity`、`shadowOffset`和`shadowRadius`
  
  这四个属性结合起来可以制作阴影效果。`shadowOpacity`默认情况下值为`0`，这意味着即便你设置了其他三个属性，只要不修改这个值，你的阴影效果就是透明的。其次，不要纠结`shadowOffset`这个决定阴影效果位置偏移的属性为什么会是`CGSize`而不是`CGPoint`。我通过下面这段代码设置的阴影效果如下：
  
		  layer.shadowColor = UIColor.grayColor().CGColor
		  layer.shadowOffset = CGSize(width: 2, height: 5)
		  layer.shadowOpacity = 1

<span><img src="/images/认识CoreAnimation/2.jpeg" width="800"></span>

- 其他属性
  
  这里包括了`transform`的仿射变换属性，相比`UIView`的同名属性，它可以设置`z`轴上的值实现更多的几何变换效果。此外还有`bound`和`frame`这些影响图层显示范围的属性，就不再多说

 CAAnimation
----

> CAAnimation的子类

开头说过，`CAAnimation`是一个封装出来的基类，其最重要的目的在于遵循两个重要的动画相关协议，所以解析动画类型要从它的子类依赖关系下手。在苹果文档中，`CAAnimation`的直接子类包括这些：

<span><img src="/images/认识CoreAnimation/3.jpeg" width="800"></span>

从图中我们可以看到存在这么三个子类:

- `CAAnimationGroup` 动画组对象，其作用是将多个`CAAnimation`动画实例组合在一起，让图层同时执行多个动画效果。在本文中不会进行更多的介绍
- `CAPropertyAnimation` 属性动画，这是很多核心动画类的父类，同时也是一个抽象的`CAAnimation`子类（这两父子都是抽象主义）他提供了对图层关键路径的属性进行动画的重要功能，在其基础上衍生的众多子类是实现动画的重要工具
- `CATransition` 过度动画类，不得不说在现今这个版本这个类的定位非常尴尬。在`CATransform3D`以及自定义转场API大行其道的这个年代，它提供的作用实在太轻微了。另一方面它还可能因为`私有api`的问题导致应用的上架失败，不过了解这个类也是可以的，在这篇[CATransition用法](http://blog.csdn.net/aluoshuai/article/details/7954862)中可以学习如何使用`CATransition`制作动画

> 类结构属性

从上面的图中我们可以看到`CAAnimation`遵循了两个协议，在其本身属性中并没有太多的属性。其中大部分的动画相关属性都是在协议中声明的，在实现中动态生成了`setter`和`getter`
 
	public class CAAnimation : NSObject, NSCoding, NSCopying, CAMediaTiming, CAAction {
	
	    public class func defaultValueForKey(key: String) -> AnyObject?
	    public func shouldArchiveValueForKey(key: String) -> Bool
	
	    public var timingFunction: CAMediaTimingFunction?
	
	    public var delegate: AnyObject?
	
	    public var removedOnCompletion: Bool
	}

通过`CAAnimation`的类结构，可以分为属性和方法两个部分，其中属性是我们需要重点关注的

- `defaultValueForKey`和`shouldArchiveValueForKey`
  
  这两个方法从名字上看就知道跟`NSCoding`协议脱不开干系，后者通过传入一个关键字对动画对象进行序列化本地存储，并且返回成功与否。然后使用相同的关键字调用前者来获取这个持久化的动画对象
  
- `timingFunction`
  
  这个是个有趣的属性，决定了动画的视觉效果。我在[从UIView动画说起](http://www.jianshu.com/p/6e326068edeb)中提到过动画的视觉效果，包括`淡入`、`淡出`等效果，这些效果用字符串表示：
  
		  public let kCAMediaTimingFunctionLinear: String
		  public let kCAMediaTimingFunctionEaseIn: String
		  public let kCAMediaTimingFunctionEaseOut: String
		  public let kCAMediaTimingFunctionEaseInEaseOut: String
		  public let kCAMediaTimingFunctionDefault: String
		  
		  let timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionEaseInEaseOut)
  
  <span><img src="/images/认识CoreAnimation/4.jpeg" width="800"></span>
  
- `delegate`
  
  在`NSObject`中实现了`CAAnimation`的回调方法，包括不限于`animationDidStart`和`animationDidStop`等方法。因此在iOS中的任何一个对象都能成为`CAAnimation`的代理人。通过这个代理我们可以在动画结束时移除动画效果等操作
  
- `removedOnCompletion `
  
  决定了动画结束之后是否将动画从相应的图层上移除，默认是`true`。由于图层动画实际上相当于障眼法（使用`CAAnimaiton`实现动画效果的时候并不会真的修改对应的属性），即是在动画结束时图层会回到动画开始的状态。通过设置这个值为`false`以及其他配置，可以避免这种事情发生

> 动画协议属性

除了`CAAnimation`本身的属性之外，另外两个协议中声明了决定动画时长、前后动画效果等关键属性：

	public protocol CAMediaTiming {
	    public var beginTime: CFTimeInterval { get set }
	    public var duration: CFTimeInterval { get set }
	    public var speed: Float { get set }
	    public var timeOffset: CFTimeInterval { get set }
	    public var repeatCount: Float { get set }
	    public var repeatDuration: CFTimeInterval { get set }
	    public var autoreverses: Bool { get set }
	    public var fillMode: String { get set }
	}

`CAMediaTiming`是一个控制动画时间的协议，提供了动画过程中的时间相关的属性，对于这些属性在[控制动画时间](http://www.cocoachina.com/programmer/20131218/7569.html)一文中讲解的非常清楚了，笔者在这里就不再一一介绍。此外，还有另一个协议`CAAction`协议：

	public protocol CAAction {
	
	    /* Called to trigger the event named 'path' on the receiver. The object
	     * (e.g. the layer) on which the event happened is 'anObject'. The
	     * arguments dictionary may be nil, if non-nil it carries parameters
	     * associated with the event. */
	
	    @available(iOS 2.0, *)
	    public func runActionForKey(event: String, object anObject: AnyObject, arguments dict: [NSObject : AnyObject]?)
	}

在动画发生之后这个方法会被调用，这个方法把将要发生的事件告诉图层，从而让图层做出对应的操作，比如渲染等。

 CAAction
----

> 显式动画

开头笔者提到过隐式动画是单独的`layer`的可动画属性发生改变时自动产生的过度动画，那么肯定就有对应的显式动画。显式动画的制作过程是创建一个动画对象，然后添加到实现动画的图层上。下面这段代码创建了一个修改`layer.position.y`的基础动画对象，然后设置动画结束值为`160`后添加到`layer`层上，动画的默认时长是`0.25`秒：

	let animation = CABasicAnimation(keyPath: "position.y")
	animation.toValue = NSNumber(float: 160)
	layer.position.y = 160
	layer.addAnimation(animation, forKey: nil)

对于动画的更详细讲解不在本篇文章的计划内，在接下来的核心动画中笔者会更加详细的介绍各式各样的`CAAnimation`子类以用于不同的动画场景。这里放上上面粘性弹窗的核心代码：

	func startAnimation() {
	    let displayLink = CADisplayLink(target: self, selector: selector(fluctAnimation(_:)))
	    displayLink.addToRunLoop(NSRunLoop.currentRunLoop(), forMode: NSRunLoopCommonModes)
	    let move = CABasicAnimation(keyPath: "position.y")
	    move.toValue = NSNumber(float: 160)
	    leftLayer.position.y = 160
	    rightLayer.position.y = 160
	    leftLayer.addAnimation(move, forKey: nil)
	    rightLayer.addAnimation(move, forKey: nil)
	
	    let spring = CASpringAnimation(keyPath: "position.y")
	    spring.damping = 15
	    spring.initialVelocity = 40
	    spring.toValue = NSNumber(float: 160)
	    centerLayer.position.y = 160
	    centerLayer.addAnimation(spring, forKey: "spring")
	}

给三个图层添加了下移的动画之后，创建`CADisplayLink`定时器来同步屏幕刷新频率更新弹出效果： 

	@objc private func fluctAnimation(link: CADisplayLink) {
	    let path = UIBezierPath()
	    path.moveToPoint(CGPointZero)
	
	    guard let _ = centerLayer.animationForKey("spring") else {
	        return
	    }
	
	    let offset = leftLayer.presentationLayer()!.position.y - centerLayer.presentationLayer()!.position.y
	    var controlY: CGFloat = 160
	    if offset < 0 {
	        controlY = centerLayer.presentationLayer()!.position.y + 30
	    } else if offset > 0 {
	        controlY = centerLayer.presentationLayer()!.position.y - 30
	    }
	
	    path.addLineToPoint(leftLayer.presentationLayer()!.position)
	    path.addQuadCurveToPoint(rightLayer.presentationLayer()!.position, controlPoint: CGPoint(x: centerLayer.position.x, y: controlY))
	    path.addLineToPoint(CGPoint(x: UIScreen.mainScreen().bounds.width, y: 0))
	    path.closePath()
	    fluctLayer.path = path.CGPath
	}

> 隐式动画发生了什么

上面说过隐式动画发生在单独的`CALayer`对象的可动画属性被改变时。如果我们这个`layer`已经存在与之绑定的`UIView`对象，那么当我们直接修改这个`layer`的属性的时候，只会瞬间从`旧值`变成`新值`的显示效果，不会有额外的效果。在`CoreAnimation`的编程指南中对此做出了解释：`UIView默认情况下禁止了layer动画，但在animate block中重新启用了它们`。这是我们看到的行为，但如果认真去挖掘这一机制的内部实现，我们会惊讶于`view`和`layer`之间协同工作的精心设计，这里就要提到`CAAction`

<span><img src="/images/认识CoreAnimation/5.jpeg" width="800"></span>

当任何一个可动画的`layer`的属性发生改变的时候，`layer`通过向它的代理人发送`actionForLayer(layer:event:)`方法来查询一个对应属性变化的`CAAction`对象，这个方法可以返回下面三个结果：

- 返回一个遵循`CAAction`的对象，这种情况下`layer`使用这个动作完成动画
- 返回一个`nil`，这样`layer`就会到其他地方继续寻找
- 返回一个`NSNull`对象，这样`layer`就会停止查找并且不执行动画

正常来说，当一个`CALayer`和`UIView`关联的时候，这个`UIView`对象会成为`layer`的代理人。因此从返回值上来说，当`layer`的属性被我们修改的时候，这个关联的`UIView`对象一般都是直接返回`NSNull`对象，而只有在`animate block`的状态下才会返回实际的动画效果，方便让图层继续查找处理动作的方案。我们通过代码来验证：

	print("===========normal call===========")
	print("\(self.view.actionForLayer(self.view.layer, forKey: "opacity"))")
	UIView.animateWithDuration(0.25) {
	    print("===========animate block call===========")
	    print("\(self.view.actionForLayer(self.view.layer, forKey: "opacity"))")
	}

控制台输出结果如下，在动画`block`中确实返回了一个`CABasicAnimation`对象来协同完成这个动画效果

	===========normal call===========
	Optional(<null>)
	===========animate block call===========
	Optional(<CABasicAnimation:0x7f8712d30c90; delegate = <UIViewAnimationState: 0x7f8712d304d0>; fillMode = both; timingFunction = easeInEaseOut; duration = 0.25; fromValue = 1; keyPath = opacity>)

通常来说处在动画代码块中的`UIView`都会返回这么一个核心动画对象，但如果返回的是`nil`，图层还有继续查找其他的动作解决方案，整个的查找过程共有四次，这个在`CALayer`的头文件中已经说明了：

<span><img src="/images/认识CoreAnimation/6.jpeg" width="800"></span>

当`layer`对象查找到了属性修改动作的动画时，就会调用`addAnimation(_:forKey:)`方法开始执行动画效果。同样的，我们继承`CALayer`对象来重写这个方法：

	class LXDActionLayer: CALayer {
	    override func addAnimation(anim: CAAnimation, forKey key: String?) {
	        print("***********************************************")
	        print("Layer will add an animation: \(anim)")
	        super.addAnimation(anim, forKey: key)
	    }
	}
	
	class LXDActionView: UIView {
	
	    override class func layerClass() -> AnyClass {
	        return LXDActionLayer.classForCoder()
	    }
	}
	
	override func viewDidLoad() {
	    super.viewDidLoad()
	
	    let actionView = LXDActionView()
	    view.addSubview(actionView)
	    print("===========normal call===========")
	    print("\(self.view.actionForLayer(self.view.layer, forKey: "opacity"))")
	    actionView.layer.opacity = 0.5
	    UIView.animateWithDuration(0.25) {
	        print("===========animate block call===========")
	        print("\(self.view.actionForLayer(self.view.layer, forKey: "opacity"))")
	    actionView.layer.opacity = 0
	}

控制台输出结果如下：

	===========normal call===========
	Optional(<null>)
	===========animate block call===========
	Optional(<CABasicAnimation:0x7f8f00eb1850; delegate = <UIViewAnimationState: 0x7f8f00eb0e90>; fillMode = both; timingFunction = easeInEaseOut; duration = 0.25; fromValue = 1; keyPath = opacity>)
	***********************************************
	Layer will add an animation: <CABasicAnimation: 0x7f8f00eb2400>

这里可能会有人有疑惑，为什么两次输出的`CABasicAnimation`的地址不一样。为了保证同一个动画对象可以作用于多个`CALayer`对象执行，在`addAnimation(_:forKey:)`方法调用的时候都会对`CAAnimation`对象进行一次`copy`操作。各位可以继承`CABasicAnimation`对象重写`copy`方法自行测试


