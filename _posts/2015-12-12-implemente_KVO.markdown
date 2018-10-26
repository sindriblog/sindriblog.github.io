---
title: 分析实现-实现KVO
date: 2015-12-12 08:00:00
tags: 分析实现
---

基于观察者设计模式，苹果实现了`notification`和`kvo`两套监听机制，两者都实现了`一对多`的监听支持。通知在设计上暴露了`notificationCenter`这个中心类，通过公开的接口和数据类型，不难猜测出其实现方式。但`KVO`仅在`NSObject`中暴露了几个接口，同时缺乏必要的中间类，文档中也只有模糊的介绍，这让人不由地对其实现机制产生兴趣。

> Automatic key-value observing is implemented using a technique called isa-swizzling... When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class ..

翻译过来就是：`KVO`是通过一种称作`isa-swizzling`的机制实现的，这个机制会在被观察对象的属性被监听时修改对象的`isa`指针，让指针指向一个中间类而非对象自身的类。

## isa
通过文档的描述，可以得出`isa`指针是`KVO`的实现机制中最为核心的变量，那么什么是`isa`指针？如果你能使用英语而非拼音来书写代码，那么一定能够明白`Objective-C`翻译过来就是`C`语言的面向对象。换句话说：

> OC的所有对象都是封装于C语言的结构体

虽然可以想象到，使用`struct`来实现面向对象的特性必然是一个十分复杂的过程，但继承的实现我们可以轻易的想象出来：在自身结构内部预留父结构体的变量。打个比方，`NSObject`的结构体为`objc_object`，存储了一个`isa`指针，假如存在子类`Person`，翻阅[objc-private](https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-private.h.auto.html)可以确定子类的结构组成：

    typedef struct objc_object *id;
    typedef struct objc_class *Class;
    struct objc_object {
        Class isa;
    };
    
    struct objc_class : objc_object {
        // Class isa;
        Class superClass;
        cache_t cache;
        class_data_bits_t bits;
        ......
    }
    
由于`id`类型属于通配类型，可以用来指向所有`OC`中的对象，根据其实现结构来看，可以说每一个`OC`对象都存在一个`isa`指针用来表示对象类型信息：

![](https://upload-images.jianshu.io/upload_images/783864-e1f69ab818b93f04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### isa-swizzling
函数`object_setClass`提供了修改`isa`指针的手段，前面已经提到了`isa`用来表示对象的所属类型，那么交换`isa`指针可以看做是修改对象的所属类型：

    /// NSObject.mm
    - (Class)class {
        return object_getClass(self);
    }

    /// code
    id obj = [NSObject new];
    NSLog(@"-class: %@, object_getClass: %@", NSStringFromClass([obj class]), NSStringFromClass(object_getClass(obj)));
    
    object_setClass(obj, [NSString class]);
    NSLog(@"-class: %@, object_getClass: %@", NSStringFromClass([obj class]), NSStringFromClass(object_getClass(obj)));
    
    /// log
    2018-01-25 09:58:46.870577+0800 Test[11398:955919] -class: NSObject, object_getClass: NSObject
    2018-01-25 09:58:46.870743+0800 Test[11398:955919] -class: NSString, object_getClass: NSString
    
方法`(+/-)(Class)class`的实现中采用`object_getclass`函数获取对象的所属类型，由于`class`方法存在被重写来误导使用者的可能性，可以直接调用`object_getclass`来获取正确的对象类型，通过这个函数可以窥见`KVO`的实现：

    - (void)test {
        id obj = [TestObj new];
        NSLog(@"-class: %@, object_getClass: %@", NSStringFromClass([obj class]), NSStringFromClass(object_getClass(obj)));
        [obj addObserver: [NSObject new] forKeyPath: @"val" options: NSKeyValueObservingOptionNew context: nil];
        NSLog(@"-class: %@, object_getClass: %@", NSStringFromClass([obj class]), NSStringFromClass(object_getClass(obj)));
        
        Class realClass = object_getClass(obj);
        NSLog(@"%@", NSStringFromClass(class_getSuperclass(realClass)));
    }
    
    // log
    2018-01-25 10:03:24.832764+0800 Test[11398:955919] -class: TestObj, object_getClass: TestObj
    2018-01-25 10:03:24.833267+0800 Test[11398:955919] -class: TestObj, object_getClass: NSKVONotifying_TestObj
    2018-01-25 10:03:24.833283+0800 Test[11398:955919] realClass's super class is: TestObj
    
在[mock in iOS](http://sindrilin.com/note/2018/07/18/mock_mechanism.html)中我曾经提到过要完全模拟一个对象包括两种手段：`inherit`或者`isa_swizzling`，结合苹果官方文档的说明，很明显苹果采用了后者。

## type-encode
`KVO`的实现基础之一是被监控对象必须拥有相应的`setter`方法，换句话说只有`ivar`的类是无法进行监控的：

    @interface UnableObservedClasss : NSobject
    {
    @public
        id _val1;
        id _val2;
    }
    
    @end

在监控过程中，`KVO`生成的新子类需要重写`setter`的实现，在属性发生修改的上下文插入执行回调的代码：

    - (void)setVal: (id)val {
        [self willChangeValueForKey: @"val"];
        [super setVal: val];
        [self didChangeValueForKey: @"val"];
    }

要实现一套通用的`KVO`机制时，是不能预设什么类型的`property`会被监控，因此如果无法区分监控属性的类型，是无法动态的去生成`setter`，我们需要使用到`type encoding`机制来协助完成这一工作。`OC`使用特定的字符编码表示某一种具体的数据类型，使用`@encode([obj class])`可以获取变量类型所对应的字符编码。下面列出官方文档中的编码对应表：

    
| 编码 | 类型 |
| --- | --- |
| c | char |
| i | int |
| s | short |
| l | long |
| q | long long |
| C | unsigned char |
| I | unsigned int |
| S | unsigned short |
| L | unsigned long |
| Q | unsigned long long |
| f | float |
| d | double |
| B | bool _Bool |
| v | void |
| * | char* |
| @ | id |
| # | Class |
| : | SEL |
| [array type] | array |
| {name=type...} | struct |
| (name=type...) | union |
| bnum | a bit field of num bits |
| ^type | pointer to type |
| ? | unknown |

对于单个`property`来说，通过`property_copyAttributeList`函数可以获取`property`的修饰符信息和类型信息，所有信息采用结构体进行映射表示：

    typedef struct {
        const char * _Nonnull name;     /// 修饰编码
        const char * _Nonnull value;    /// 具体内容
    } objc_property_attribute_t;
    
有两个重要的修饰编码：`T`表示类型编码，通过匹配编码表确认类型；`S`表示属性含有`setter`，可以动态的生成`KVO`的方法

## 实现
参照`YYModel`对于属性`setter`的封装实现：

    /// 获取监控的属性
    objc_property_t getKVOProperty(Class cls, NSString *keyPath) {
        if (!keyPath || !cls) {
            return NULL;
        }
        
        objc_property_t res = NULL;
        unsigned int count = 0;
        const char *property_name = keyPath.UTF8String;
        objc_property_t *properties = class_copyPropertyList(cls, &count);
        
        for (unsigned int idx = 0; idx < count; idx++) {
            objc_property_t property = properties[idx];
            if (strcmp(property_name, property_getName(property)) == 0) {
                res = property;
                break;
            }
        }
        free(properties);
        return res;
    }
    
    /// 检测属性是否存在setter方法
    BOOL ifPropertyHasSetter(objc_property_t property) {
        BOOL res = NO;
        unsigned int attrCount;
        objc_property_attribute_t *attrs = property_copyAttributeList(property, &attrCount);
        
        for (unsigned int idx = 0; idx < attrCount; idx++) {
            if (attrs[idx].name[0] == 'S') {
                res = YES;
            }
        }
        free(attrs);
        return res;
    }
    
    /// 获取属性的数据类型
    YYEncodingType getPropertyType(objc_property_t) {
        unsigned int attrCount;
        YYEncodingType type = YYEncodingTypeUnknown;
        objc_property_attribute_t *attrs = property_copyAttributeList(property, &attrCount);
        
        for (unsigned int idx = 0; idx < attrCount; idx++) {
            if (attrs[idx].name[0] == 'T') {
                type = YYEncodingGetType(attrs[idx].value);
            }
        }
        free(attrs);
        return type;
    }
    
    /// 根据setter名称获取属性名
    NSString *getPropertyNameFromSelector(SEL selector) {
        NSString *selName = [NSStringFromSelector(selector) substringFromIndex: 3];
        NSString *firstAlpha = [[selName substringToIndex: 1] lowercaseString];
        return [selName stringByReplacingCharactersInRange: NSMakeRange(0, 1) withString: firstAlpha];
    }
    
    /// 根据属性名获取setter名称
    SEL getSetterFromKeyPath(NSString *keyPath) {
        NSString *firstAlpha = [[keyPath substringToIndex: 1] uppercaseString];
        NSString *selName = [NSString stringWithFormat: @"set%@", [keyPath stringByReplacingCharactersInRange: NSMakeRange(0,  1) withString: firstAlpha]];
        return NSSelectorFromString(selName);
    }
    
    /// 设置bool属性的kvo setter
    static void setBoolVal(id self, SEL _cmd, BOOL val) {
        NSString *name = getPropertyNameFromSelector(_cmd);
        void (*objc_msgSendKVO)(void *, SEL, NSString *) = (void *)objc_msgSend;
        void (*objc_msgSendSuperKVO)(void *, SEL, BOOL) = (void *)objc_msgSendSuper;
        
        objc_msgSendKVO(self, @selector(willChangeValueForKey:), val);
        objc_msgSendSuperKVO(self, _cmd, val);
        objc_msgSendKVO(self, @selector(didChangeValueForKey:), val);
    }

    /// KVO实现
    static void addObserver(id observedObj, id observer, NSString *keyPath) {
        objc_property_t observedProperty = getKVOProperty([observedObj class], keyPath);
        if (!ifPropertyHasSetter(observedProperty)) {
            return;
        }
        
        NSString *kvoClassName = [@"SLObserved_" stringByAppendString: NSStringFromClass([observedObj class])];
        Class kvoClass = NSClassFromString(kvoClassName);
        if (!kvoClass)) {
            kvoClass = objc_allocateClassPair([observedObj class], kvoClassName.UTF8String, NULL);
            
            Class(^classBlock)(id) = ^Class(id self) {
                return class_getSuperclass([self class]);
            };
            class_addMethod(kvoClass, @selector(class), imp_implementationWithBlock(classBlock), method_getTypeEncoding(class_getMethodImplementation([observedObj class], @selector(class))));
            objc_registerClassPair(kvoClass);
        }
        
        YYEncodingType type = getPropertyType(observedProperty);
        SEL setter = getSetterFromKeyPath(observedProperty);
        switch (type) {
            case YYEncodingTypeBool: {
                class_addMethod(kvoClass, setter, (IMP)setBoolVal, method_getTypeEncoding(class_getMethodImplementation([observedObj class], setter)));
            }   break;
            ......
        }
    }


