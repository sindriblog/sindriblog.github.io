---
title: 开发笔记-mock in iOS
date: 2018-07-18 08:00:00
tags:
- 开发笔记
---

在面向对象编程中，有个非常有趣的概念叫做`duck type`，意思是如果有一个走路像鸭子、游泳像鸭子，叫声像鸭子的东西，那么它就可以被认为是鸭子。这意味着当我们需要一个鸭子对象时，可以通过`instantiation`或者`interface`两种机制来提供鸭子对象：

    @interface Duck : NSObject

    @property (nonatomic, assign) CGFloat weigh;

    - (void)walk;
    - (void)swim;
    - (void)quack;

    @end

    /// instantiation
    id duckObj = [[Duck alloc] init];
    [TestCase testWithDuck: duckObj];

    /// interface
    @protocol DuckType

    - (void)walk;
    - (void)swim;
    - (void)quack;
    - (CGFloat)weigh;
    - (void)setWeigh: (CGFloat)weigh;

    @end

    @interface MockDuck : NSObject<DuckType>
    @end

    id duckObj = [[MockDuck alloc] init];
    [TestCase testWithDuck: duckObj];

后者定义了一套鸭子接口，模仿出了一个`duck type`对象，虽然对象是模拟的，但这并不阻碍程序的正常执行，这种设计思路，可以被称作`mock`

>  通过制造模拟真实对象行为的假对象，来对程序功能进行测试或调试

## interface和mock
虽然上面通过`interface`的设计实现了`mock`的效果，但两者并不能划上等号。从设计思路上来说，`interface`是抽象出一套行为接口或者属性，且并不关心实现者是否存在具体实现上的差异。而`mock`需要模拟对象和真实对象两者具有相同的行为和属性，以及一致的行为实现：

    /// interface
    一个测试工程师进了一间酒吧点了一杯啤酒
    一个开发工程师进了一间咖啡厅点了一杯咖啡
    
    /// mock
    一个测试工程师进了一间酒吧点了一杯啤酒
    一个模拟的测试工程师进了一间酒吧点了一杯啤酒

从实现上来说，虽然`interface`可以通过抽象出真实对象所有的行为和属性来完成对真实对象的百分百还原，但这样就违背了`interface`应只提供一系列相同功能接口的原则，因此`interface`更适用于模块解耦、功能扩展相关的工作。而`mock`由于要求模拟对象对真实对象百分百的`copy`，更多的应用在调试、测试等方面的工作

## 如何实现mock
个人把`mock`根据模拟程度分为行为模拟和完全模拟两种情况，对于真实对象的模拟，总共包括四种方式：

- `inherit`
- `interface`
- `forwarding`
- `isa_swizzling`

### 行为模拟
行为模拟追求的是对真实对象的核心行为进行还原。由于`OC`的消息处理机制，因此无论是`interface`的接口扩展还是`forwarding`的转发处理都可以完成对真实对象的模拟：

    /// interface
    @interface InterfaceDuck : NSObject<DuckType>
    @end
    
    /// forwarding
    @interface ForwardingDuck : NSObject
    
    @property (nonatomic, strong) Duck *duck;

    @end

    @implementation MockDuck

    - (id)forwardingTargetForSelector: (SEL)selector {
        return _duck;
    }

    @end

`interface`跟`forwarding`的区别在于后者的真正处理者可以是真实对象本身，不过由于`forwarding`不一定非要转发给真实对象处理，所以二者既可以是行为模拟，也可以是完全模拟。但更多时候，两者是`duck type`

### 完全模拟
完全模拟要求以假乱真，在任何情况下模拟对象可以表现的跟真实对象无差别化：

    @interface MockDuck : Duck
    @end

    /// inherit
    MockDuck *duck = [[MockDuck alloc] init];
    [TestCase testWithDuck: duck];

    /// isa_swizzling
    Duck *duck = [[Duck alloc] init];
    object_setClass(duck, [MockDuck class]);
    [TestCase testWithDuck: duck];

虽然`inherit`和`isa_swizzling`两种方式的行为没有任何差别，但是后者更像是`借用`了子类的所有属性、结构，而只呈现`Duck`的行为。但在单元测试中的`mock`，由于并不存在直接进行`isa_swizzling`的真实对象，还需要动态的生成`class`来完成模拟对象的构建：

    Class MockClass = objc_allocateClassPair(RealClass, RealClassName, 0);
    objc_registerClassPair(MockClass);
    
    for (Selector s in getClassSelectors(RealClass)) {
        Method m = class_getInstanceMethod(RealClass, s);
        class_addMethod(MockClass, s, method_getImplementation(m), method_getTypeEncoding(m));
    }
    id mockObj = [[MockClass alloc] init];
    [TestCase testWithObj: mockObj];

### 结构模拟
结构模拟是一种威力和破坏能力同样强大的`mock`方式，由于数据结构最终采用二进制存储，结构模拟尝试构建整个真实对象的二进制结构布局，然后修改结构内变量。同时，结构模拟并不要求必须掌握对象的准确布局信息，只要清楚我们需要修改的数据位置就行了。譬如`OC`的`block`实际上是一个可变长度的结构体，结构体的大小会随着捕获的变量数量增大，但是前`32`位的存储信息是固定的，其结构体如下：

    struct Block {
        void *isa;
        int flags;
        int reserved;
        void (*invoke)(void *, ...);
        struct BlockDescriptor *descriptor;
        /// catched variables
    };

其中`invoke`指针指向了其`imp`的函数地址，只要修改这个指针值，就能改变`block`的行为：

    struct MockBlock {
        ...
    };

    void printHelloWorld(void *context) {
        printf("hello world\n");
    };

    dispatch_block_t block = ^{
        printf("I'm block!\n");
    };
    struck MockBlock *mock = (__bridge struct MockBlock *)block;
    mock->invoke(NULL);
    mock->invoke = printHelloWorld;
    block();

通过`mock`真实对象的结构布局来获取真实对象的行为，甚至修改行为，虽然这种做法非常强大，但如果因为系统版本的差异导致对象的结构布局存在差异，或者获取的布局信息并不准确，就会破坏数据本身，导致意外的程序错误

## 什么时候用mock
从个人开发经历来看，如果有以下情况，我们可以考虑使用`mock`来替换真实对象：

- `类型缺失运行环境`
- `结果依赖于异步操作`
- `真实对象对外不可见`

其中前两者更多发生在单元测试中，而后者多与调试工作相关

### 类型缺失运行环境
`NSUserDefaults`会将数据以`key-value`的对应格式存储在沙盒目录下，但在单元测试的环境下，程序并没有编程成二进制包，因此`NSUserDefaults`无法被正常使用，因此使用`mock`可以还原测试场景。通常会选择`OCMock`来完成单元测试的`mock`需求：

    - (void)testUserDefaultsSave: (NSUserDefaults *)userDefaults {
          [userDefaults setValue: @"value" forKey: @"key"];
          XCTAssertTrue([[userDefaults valueForKey: @"key"] isEqualToString: @"value"])
    }

    id userDefaultsMock = OCMClassMock([NSUserDefaults class]);
    OCMStub([userDefaultsMock valueForKey: @"key"]).andReturn(@"value");
    [self testUserDefaultsSave: userDefaultsMock];

实际上在单元测试中，与沙盒相关的`IO`类都几乎处于不可用状态，因此`mock`这样的数据结构可以很好的提供对沙盒存储功能的支持

### 结果依赖于异步操作
`XCTAssert`为异步操作提供了一个延时接口，当然并没有卵用。异步处理往往是单元测试的杀手，`OCMock`同样提供了对于异步的接口支持：

    - (void)requestWithData: (NSString *)data complete: (void(^)(NSDictionary *response))complete;
    
    OCMStub([requestMock requestWithData: @"xxxx" complete: [OCMArg any]]).andDo(^(NSInvocation *invocation) {
        /// invocation是request方法的封装
        void (^complete)(NSDictionary *response);
        [invocation getArgument: &complete atIndex: 3];

        NSDictionary *response = @{
                                  @"success": @NO,
                                  @"message": @"wrong data"
                                  };
        complete(response);
    });

抛开已有的第三方工具，通过消息转发机制也可以实现一个处理异步测试的工具：

    @interface AsyncMock : NSProxy {
        id _callArgument;
    }

    - (instancetype)initWithAsyncCallArguments: (id)callArgument;

    @end

    @implementation AsyncMock

    - (void)forwardInvocation: (NSInvocation *)anInvocation {
        id argument = nil;
        for (NSInteger idx = 2; idx <anInvocation.methodSignature.numberOfArguments; idx++) {
            [anInvocation getArgument: &argument atIndex: idx];
            if ([[NSString stringWithUTF8String: @encode(argument)] hasPrefix: @"@?"]) {
                break;
            }
        }
        if (argument == nil) {
            return;
        }

        void (^block)(id obj)  = argument;
        block(_callArgument;)
    }

    @end

    NSDictionary *response = @{
                              @"success": @NO,
                              @"message": @"wrong data"
                              };
    id requestMock = [[AsyncMock alloc] initWithAsyncCallArgument: response];
    [requestMock requestWithData: @"xxxx" complete: ^(id obj) {
        /// do something when request complete
    }];

转发的最后一个阶段会将消息包装成`NSInvocation`对象，`invocation`提供了遍历获取调用参数的信息，通过`@encode()`对参数类型进行判断，获取回调`block`并且调用

### 真实对象对外不可见
真实对象对外不可见存在两种情况：

- `结构不可见`
- `结构实例均不可见`

几乎在所有情况下我们遇到的都是`结构不可见`，比如私有类、私有结构等，上文中提到的`block`结构体就是最明显的例子，通过`clang`命令重写类文件基本可以得到这类对象的结构内部。由于上文已经展示过`block`的布局模拟，这里就不再多说

    clang -rewrite-objc xxx.m

而后者比较特殊，无论是结构布局，还是实例对象，我们都无法获取到。打个比方，我需要统计应用编译包的二进制段的信息，通过使用`hopper`工具可以得到`objc_classlist_DATA`段的情况：

![](https://upload-images.jianshu.io/upload_images/783864-7956de594ccd2878.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于此时没有任何的真实对象和结构参考，只能知道每一个`__objc_data`的长度是`72`字节。因此这种情况下需要先模拟出等长于二进制数据的结构体，然后通过输出`16`进制数据来匹配数据段的布局信息：

    struct __mock_binary {
        uint vals[18];
    };

    NSMutableArray *binaryStrings = @[].mutableCopy;
    for (int idx = 0; idx <18; idx++) {
        [binaryStrings appendString: [NSString stringWithFormat: @"%p", (void *)binary->vals[idx]]];
    }
    NSLog(@"%@", [binaryStrings componentsJoinedByString: @"  "]);

通过分析`16`进制段数据，结合`hopper`得出的数据段信息，可以绘制出真实对象的布局信息，然后采用结构模拟的方式构建模拟的结构体：

    struct __mock_objc_data {
        uint flags;
        uint start;
        uint size;
        uint unknown;
        uint8_t *ivarlayouts;
        uint8_t *name;
        uint8_t *methods;
        uint8_t *protocols;
        uint8_t *ivars;
        uint8_t *weaklayouts;
        uint8_t *properties;
    };

    struct __mock_objc_class {
        uint8_t *meta;
        uint8_t *super;
        uint8_t *cache;
        uint8_t *vtable;
        struct __mock_objc_data *data;
    };

    struct load_command *cmds = (struct load_command *)sizeof(struct mach_header_64);
    for (uint idx = 0; idx <header.ncmds; idx++, cmds = (struct load_command *)(uint8_t *)cmds + cmds->cmdsize) {
        struct segment_command_64 *segCmd = (struct segment_command_64 *)cmds;
        struct section_64 *sections = (struct section_64 *)((uint8_t *)cmds +sizeof(struct segment_command_64));

        uint8_t *secPtr = (uint8_t *)section->offset;
        struct __mock_objc_class *objc_class = (struct __mock_objc_class *)secPtr;
        struct __mock_objc_data *objc_data = objc_class->data;
        printf("%s in objc_classlist_DATA\n", objc_data->name);
        ......
    }

上述代码已做简化展示。实际上遍历`machO`需要将二进制文件载入内存，还要考虑`hopper`加载跟自己手动加载的地址偏移差，最终求出一个正确的地址值。在整个遍历过程中，除了`header`和`command`等结构是系统暴露的之后，其他存储对象都需要去查看`hopper`加上进制数值进行推导，最终`mock`出结构完成工作

## 总结
`mock`并不是一种特定的操作或者编程手段，它更像是一种剖析工程细节来解决特殊环境下难题的解决思路。无论如何，如果我们想要继续在开发领域上继续深入，必然要学会从更多的角度和使用更多的工具来理解和解决开发上的难题，而`mock`绝对是一种值得学习的开发思想

![关注我的公众号获取更新信息](https://github.com/sindriblog/sindriblog.github.io/blob/master/assets/images/wechat_code.jpg?raw=true)


