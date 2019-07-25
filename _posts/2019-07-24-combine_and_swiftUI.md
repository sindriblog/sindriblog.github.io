---
title: Combine和SwiftUI
date: 2019-07-24 08:00
tags:
- WWDC
---

尽管今年的`WWDC`已经落幕，但在过去的一个多月时间，苹果给`iOS`开发者带来了许多惊喜，其中堪称最重量级的当属`SwiftUI`和`Combine`两大新框架

在更早之前，由于缺少系统层的声明式`UI`语言，在`iOS`系统上的`UI`开发对于开发者而言，并不友善，而从`iOS13`开始，开发者们终于可以摆脱落后的布局系统，拥抱更简洁高效的开发新时代。与`SwiftUI`配套发布的响应式编程框架`Combine`提供了更优美的开发方式，这也意味着`Swift`真正成为了`iOS`开发者们必须学习的语言。本文基于`Swift5.1`版本，介绍`SwiftUI`是如何通过结合`Combine`完成数据绑定

## SwiftUI
![](https://upload-images.jianshu.io/upload_images/783864-c9e79750dcdb192c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先来个例子，假如我们要实现上图的登陆界面，按照以往使用`UIKit`进行开发，那么我们需要：

- 创建一个`UITextField`，用于输入账户
- 创建一个`UITextField`，用于输入密码
- 创建一个`UIButton`，设置点击事件将前两个`UITextField`的文本作为数据请求

而在使用`SwiftUI`进行开发的情况下，代码如下：

    public struct LoginView : View {
        @State var username: String = ""
        @State var password: String = ""
        
        public var body: some View {
            VStack {
                TextField($username, placeholder:  Text("Enter username"))
                    .textFieldStyle(.roundedBorder)
                    .padding([.leading, .trailing], 25)
                    .padding([.bottom], 15)
                SecureField($password, placeholder: Text("Enter password"))
                    .textFieldStyle(.roundedBorder)
                    .padding([.leading, .trailing], 25)
                    .padding([.bottom], 30)
                Button(action: {}) {
                    Text("Sign In")
                        .foregroundColor(.white)
                    }.frame(width: 120, height: 40)
                    .background(Color.blue)
            }
        }
    }
    
在`SwiftUI`中，使用`@State`修饰的属性会在发生改变的时候通知绑定的`UI`控件强制刷新渲染，这种新命名归功于新的`PropertyWrapper`机制。可以看到`SwiftUI`的控件命名上和`UIKit`几乎保持一致的，下表是两个标准库上的`UI`对应表：

    
| SwiftUI | UIKit |
| --- | --- |
| Text | UILabel / NSAttributedString |
| TextField | UITextField |
| SecureField | UITextField with isSecureTextEntry |
| Button | UIButton |
| Image | UIImageView |
| List | UITableView |
| Alert | UIAlertView / UIAlertController |
| ActionSheet | UIActionSheet / UIAlertController |
| NavigationView | UINavigationController |
| HStack | UIStackView with horizatonal |
| VStack | UIStackView with vertical |
| Toggle | UISwitch |
| Slider | UISlider |
| SegmentedControl | UISegmentedControl |
| Stepper | UIStepper |
| DatePicker | UIDatePicker |

   
### View
    @available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
    public protocol View : _View {
    
        /// The type of view representing the body of this view.
        ///
        /// When you create a custom view, Swift infers this type from your
        /// implementation of the required `body` property.
        associatedtype Body : View
    
        /// Declares the content and behavior of this view.
        var body: Self.Body { get }
    }

虽然在`SwiftUI`中使用`View`来表示可视控件，但实际上大相径庭，`View`是一套容器协议，不展示任何内容，只定义了一套视图的交互、布局等接口。`UI`控件需要实现协议中的`body`返回需要展示的内容。另外`View`还扩展了`Combine`响应式编程的订阅接口：
    
    @available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
    extension View {
    
        /// Adds an action to perform when the given publisher emits an event.
        ///
        /// - Parameters:
        ///   - publisher: The publisher to subscribe to.
        ///   - action: The action to perform when an event is emitted by
        ///     `publisher`. The event emitted by publisher is passed as a
        ///     parameter to `action`.
        /// - Returns: A view that triggers `action` when `publisher` emits an
        ///   event.
        public func onReceive<P>(_ publisher: P, perform action: @escaping (P.Output) -> Void) -> SubscriptionView<P, Self> where P : Publisher, P.Failure == Never
    
        /// Adds an action to perform when the given publisher emits an event.
        ///
        /// - Parameters:
        ///   - publisher: The publisher to subscribe to.
        ///   - action: The action to perform when an event is emitted by
        ///     `publisher`.
        /// - Returns: A view that triggers `action` when `publisher` emits an
        ///   event.
        public func onReceive<P>(_ publisher: P, perform action: @escaping () -> Void) -> SubscriptionView<P, Self> where P : Publisher, P.Failure == Never
    }
    
### @State
    @available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
    @propertyDelegate public struct State<Value> : DynamicViewProperty, BindingConvertible {
    
        /// Initialize with the provided initial value.
        public init(initialValue value: Value)
    
        /// The current state value.
        public var value: Value { get nonmutating set }
    
        /// Returns a binding referencing the state value.
        public var binding: Binding<Value> { get }
    
        /// Produces the binding referencing this state value
        public var delegateValue: Binding<Value> { get }
    
        /// Produces the binding referencing this state value
        /// TODO: old name for storageValue, to be removed
        public var storageValue: Binding<Value> { get }
    }
    
`Swift5.1`的新特性之一，开发者可以将变量的`IO`实现封装成通用逻辑，用关键字~~`@propertyDelegate`~~（更新于`beta4`版本，使用`@propertyWrapper`替代）修饰读写逻辑，并以`@wrapperName var variable`的方式封装变量。以[WWDC Session 415](https://developer.apple.com/videos/play/wwdc2019/415/)视频中的例子实现对变量`copy-on-write`的封装：

    @propertyWrapper
    public struct DefensiveCopying<Value: NSCopying> {
        private var storage: Value
        
        public init(initialValue value: Value) {
            storage = value.copy() as! Value
        }
        
        public var wrappedValue: Value {
            get { storage }
            set {
                storage = newValue.copy() as! Value
            }
        }
        
        ///  beta4版本更新，必须声明projectedValue后才能使用$variable的方式访问Wrapper<Value>
        ///  beta3版本使用wrapperValue命名
        public var projectedValue: DefensiveCopying<Value> {
            get { self }
        }
    }
    
    public struct Person {
        @DefensiveCopying(initialValue: "")
        public var name: NSString
    }
    
另外以`PropertyWrapper`封装的变量，会~~默认生成一个命名为`$name`的`DefensiveCopying<String>`类型的变量~~，更新后会默认生成`_name`命名的`Wrapper<Value>`类型参数，或声明关键变量`wrapperValue/projectedValue`后生成可访问的`$name`变量，下面两种值访问操作是相同的：

    extension Person {
        func visitName() {
            printf("name: \(name)")
            printf("name: \($name.value)")
        }
    }

## Combine
> Customize handling of asynchronous events by combining event-processing operators.

引用官方文档的描述，`Combine`是一套通过组合变换事件操作来处理异步事件的标准库。事件执行过程的关系包括：被观察者`Observable`和观察者`Observer`，在`Combine`中对应着`Publisher`和`Subscriber`

### 异步编程
很多开发者认为异步编程会开辟线程执行任务，多数时候程序在异步执行时确实也会创建线程，但是这种理解是不正确的，同步编程和异步编程的区别只在于程序是否会堵塞等待任务执行完毕，下面是一段无需额外线程的异步编程实现代码：

    class TaskExecutor {
        static let instance = TaskExecutor()
        private var executing: Bool = false
        private var tasks: [() -> ()] = Array()
        private var queue: DispatchQueue = DispatchQueue.init(label: "SerialQueue")
        
        func pushTask(task: @escaping () -> ()) {
            tasks.append(task)
            if !executing {
                execute()
            }
        }
        
        func execute() {
            executing = true
            let executedTasks = tasks
            tasks.removeAll()
            executedTasks.forEach {
                $0()
            }
            if tasks.count > 0 {
                execute()
            } else {
                executing = false
            }
        }
    }
    
    TaskExecutor.instance.execute()
    TaskExecutor.instance.pushTask {
        print("abc")
        TaskExecutor.instance.pushTask {
            print("def")
        }
        print("ghi")
    }
    

### 单向流动
如果`A`事件会触发`B`事件，反之不成立，可以认为两个事件是单向的，好比说`我饿了，所以我去吃东西`，但不会是`我去吃东西，所以我饿了`。在编程中，如果数据流动能够保证单向，会让程序变得更加简单。举个例子，下面是一段非单向流动的常见`UI`代码：

    func tapped(signIn: UIButton) {
        LoginManager.manager.signIn(username, password: password) { (err) in
            guard err == nil else {
                ERR_LOG("sign in failed: \(err)")
                return
            }
            UserManager.manager.switch(to: username)
            MainPageViewController.enter()
        }
    }

在这段代码中，`Action`实际上会等待`State/Data`完成后，去更新`View`，`View`会再去访问数据更新状态，这种逻辑会让数据在不同事件模块中随意流动，易读性和可维护性都会变得更差：

![](https://upload-images.jianshu.io/upload_images/783864-28de8dbc69731179.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而一旦事件之间的流动采用了异步编程的方式来处理，发出事件的人不关心等待事件的处理，无疑能让数据的流动变得更加单一，`Combine`的意义就在于此。`SwiftUI`与其结合来控制业务数据的单向流动，让开发复杂度大大降低：

![来自淘宝技术](https://upload-images.jianshu.io/upload_images/783864-623c23d1c20ce511?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Publisher

    @available(OSX 10.15, iOS 13.0, tvOS 13.0, watchOS 6.0, *)
    public protocol Publisher {
    
        /// The kind of values published by this publisher.
        associatedtype Output
    
        /// The kind of errors this publisher might publish.
        ///
        /// Use `Never` if this `Publisher` does not publish errors.
        associatedtype Failure : Error
    
        /// This function is called to attach the specified `Subscriber` to this `Publisher` by `subscribe(_:)`
        ///
        /// - SeeAlso: `subscribe(_:)`
        /// - Parameters:
        ///     - subscriber: The subscriber to attach to this `Publisher`.
        ///                   once attached it can begin to receive values.
        func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
    }
    
`Publisher`定义了发布相关的两个信息：`Output`和`Failure`，对应事件输出值和失败处理两种情况，以及提供了`receive(subscriber:)`接口注册事件订阅者。在`iOS13`之后，苹果基于`Foundation`标准库实现了很多`Combine`的响应式接口，包括：

- `URLSessionTask`可以在请求完成或者请求出错时发出消息

- `NotificationCenter`新增响应式编程接口

以官方的`NotificationCenter`扩展为例创建一个登录操作的`Publisher`：

    extension NotificationCenter {
        struct Publisher: Combine.Publisher {
            typealias Output = Notification
            typealias Failure = Never
            init(center: NotificationCenter, name: Notification.Name, Object: Any? = nil)
        }
    }
    
    let signInNotification = Notification.Name.init("user_sign_in")
    
    struct SignInInfo {
        let username: String
        let password: String
    }
    
    let signInPublisher = NotificationCenter.Publisher(center: .default, name: signInNotification, object: nil)
    
另外还需要注意的是：`Self.Output == S.Input`限制了`Publisher`和`Subscriber`之间的数据流动必须保持类型一致，大多数时候总是很难维持一致性的，所以`Publisher`同样提供了`map/compactMap`的高阶函数对输出值进行转换：

    /// Subscriber只接收用户名信息
    let signInPublisher = NotificationCenter.Publisher(center: .default, name: signInNotification, object: nil)
        .map {
            return ($0 as? SignInfo)?.username ?? "unknown"
        }

### Subscriber
    @available(OSX 10.15, iOS 13.0, tvOS 13.0, watchOS 6.0, *)
    public protocol Subscriber : CustomCombineIdentifierConvertible {
    
        /// The kind of values this subscriber receives.
        associatedtype Input
    
        /// The kind of errors this subscriber might receive.
        ///
        /// Use `Never` if this `Subscriber` cannot receive errors.
        associatedtype Failure : Error
    
        /// Tells the subscriber that it has successfully subscribed to the publisher and may request items.
        ///
        /// Use the received `Subscription` to request items from the publisher.
        /// - Parameter subscription: A subscription that represents the connection between publisher and subscriber.
        func receive(subscription: Subscription)
    
        /// Tells the subscriber that the publisher has produced an element.
        ///
        /// - Parameter input: The published element.
        /// - Returns: A `Demand` instance indicating how many more elements the subcriber expects to receive.
        func receive(_ input: Self.Input) -> Subscribers.Demand
    
        /// Tells the subscriber that the publisher has completed publishing, either normally or with an error.
        ///
        /// - Parameter completion: A `Completion` case indicating whether publishing completed normally or with an error.
        func receive(completion: Subscribers.Completion<Self.Failure>)
    }
    
`Subscriber`定义了一套`receive`接口用来接收`Publisher`发送的消息，一个完整的订阅流程如下图：

![](https://upload-images.jianshu.io/upload_images/783864-9b5692504c959605.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在订阅成功之后，`receive(subscription:)`会被调用一次，其类型如下：

    @available(OSX 10.15, iOS 13.0, tvOS 13.0, watchOS 6.0, *)
    public protocol Subscription : Cancellable, CustomCombineIdentifierConvertible {
    
        /// Tells a publisher that it may send more values to the subscriber.
        func request(_ demand: Subscribers.Demand)
    }
    
`Subscription`可以认为是单次订阅的会话，其实现了`Cancellable`接口允许`Subscriber`中途取消订阅，释放资源。基于上方的`NotificationCenter`代码，完成`Subscriber`的接收部分：

    func registerSignInHandle() {
        let signInSubscriber = Subscribers.Assign.init(object: self.userNameLabel, keyPath: \.text)
        signInPublisher.subscribe(signInSubscriber)
    }
    
    func tapped(signIn: UIButton) {
        LoginManager.manager.signIn(username, password: password) { (err) in
            guard err == nil else {
                ERR_LOG("sign in failed: \(err)")
                return
            }
            let info = SignInfo(username: username, password: password)
            NotificationCenter.default.post(name: signInNotification, object: info)
        }
    }

## Combine与UIKit
得力于`Swift5.1`的新特性，基于`PropertyWrapper`和`Combine`标准库，可以让`UIKit`同样具备绑定数据流动的能力，预设代码如下：

    class ViewController: UIViewController {
    
        @Publishable(initialValue: "")
        var text: String
        
        let textLabel = UILabel.init(frame: CGRect.init(x: 100, y: 120, width: 120, height: 40))
    
        override func viewDidLoad() {
            super.viewDidLoad()
            textLabel.bind(text: $text)
            
            let button = UIButton.init(frame: CGRect.init(x: 100, y: 180, width: 120, height: 40))
            button.addTarget(self, action: #selector(tapped(button:)), for: .touchUpInside)
            button.setTitle("random text", for: .normal)
            button.backgroundColor = .blue
            
            view.addSubview(textLabel)
            view.addSubview(button)
        }
        
        @objc func tapped(button: UIButton) {
            text = String(arc4random() % 101)
        }
    }
    
每次点击按钮的时候生成随机数字符串，然后`textLabel`自动更新文本

### Publishable
字符串在发生改变的时候需要更新绑定的`label`，在这里使用`PassthroughSubject`类对输出值类型做强限制，其结构如下：

    @available(OSX 10.15, iOS 13.0, tvOS 13.0, watchOS 6.0, *)
    final public class PassthroughSubject<Output, Failure> : Subject where Failure : Error {
    
        public init()
    
        /// This function is called to attach the specified `Subscriber` to this `Publisher` by `subscribe(_:)`
        ///
        /// - SeeAlso: `subscribe(_:)`
        /// - Parameters:
        ///     - subscriber: The subscriber to attach to this `Publisher`.
        ///                   once attached it can begin to receive values.
        final public func receive<S>(subscriber: S) where Output == S.Input, Failure == S.Failure, S : Subscriber
    
        /// Sends a value to the subscriber.
        ///
        /// - Parameter value: The value to send.
        final public func send(_ input: Output)
    
        /// Sends a completion signal to the subscriber.
        ///
        /// - Parameter completion: A `Completion` instance which indicates whether publishing has finished normally or failed with an error.
        final public func send(completion: Subscribers.Completion<Failure>)
    }

`Publishable`的实现代码如下（7.25更新）：

    @propertyWrapper
    public struct Publishable<Value: Equatable> {
        private var storage: Value
        var publisher: PassthroughSubject<Value?, Never>
        
        public init(initialValue value: Value) {
            storage = value
            publisher = PassthroughSubject<Value?, Never>()
            Publishers.AllSatisfy
        }
        
        public var wrappedValue: Value {
            get { storage }
            set {
                if storage != newValue {
                    storage = newValue
                    publisher.send(storage)
                }
            }
        }

        public var projectedValue: Publishable<Value> {
            get { self }
        }
    }
    
### UI extensions
通过`extension`对控件进行扩展支持属性绑定：

    extension UILabel {
    
        func bind(text: Publishable<String>) {
            let subscriber = Subscribers.Assign.init(object: self, keyPath: \.text)
            text.publisher.subscribe(subscriber)
            self.text = text.value
        }
    }

这里需要注意的是，创建的`Subscriber`会被系统的`libswiftCore`持有，在控制器生命周期结束时，如果不能及时的`cancel`掉所有的`subscriber`，会导致内存泄漏：

    func freeBinding() {
        subscribers?.forEach {
            $0.cancel()
        }
        subscribers?.removeAll()
    }
    
最后放上运行效果：

![](https://upload-images.jianshu.io/upload_images/783864-0dc6df8edf9242a9.gif?imageMogr2/auto-orient/strip)

## 其他
从今年`wwdc`发布的新内容，不难看出苹果的野心，由于`Swift`本身就是一门特别适合编写`DSL`的语言，而在`iOS13`上新增的两个标准库让项目的开发成本和维护成本变得更低的特点。由于其极高的可读性，开发者很容易就习惯新的标准库。目前`SwiftUI`实时用到了`UIKit`、`CoreGraphics`等库，可以看做是基于这些库的抽象封装层，随着后续`Swift`的普及度，苹果底层可以换掉`UIKit`独立存在，甚至实现跨平台的大前端一统。当然目前苹果上的大前端尚早，不过未来可期

## 参考阅读
[SwiftUI Session](https://developer.apple.com/videos/wwdc2019/?q=SwiftUI)

[Property Wrappers](https://www.avanderlee.com/swift/property-wrappers/)

[Combine入门导读](https://icodesign.me/posts/swift-combine/)

[新晋网红SwiftUI](https://mp.weixin.qq.com/s/x_jFcKeXSbtdK0CnfayFsw)


