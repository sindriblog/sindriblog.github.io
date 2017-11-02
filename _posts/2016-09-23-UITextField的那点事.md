---
layout: post
title: UITextField的那点事
categories: Tips
tags: Tips
author: SindriLin
---

* content
{:toc}


`UITextField`被用作项目中获取用户信息的重要控件，但是在实际应用中存在的不少的坑：修改`keyboardType`来限制键盘的类型，却难以限制第三方键盘的输入类型；在代理中限制了输入长度以及输入的文本类型，但是却抵不住中文输入的联想；键盘弹起时遮住输入框，需要接收键盘弹起收回的通知，然后计算坐标实现移动动画。

对于上面这些问题，苹果提供给我们文本输入框的同时并不提供解决方案，因此本文将使用`category+runtime`的方式解决上面提到的这些问题，本文假设读者已经清楚从`UITextField`成为第一响应者到结束编辑过程中的事件调用流程。

输入限制
----
最常见的输入限制是手机号码以及金额，前者文本中只能存在纯数字，后者文本中还能包括小数。笔者暂时定义了三种枚举状态用来表示三种文本限制：

    typedef NS_ENUM(NSInteger, LXDRestrictType)
    {
        LXDRestrictTypeOnlyNumber = 1,      ///< 只允许输入数字
        LXDRestrictTypeOnlyDecimal = 2,     ///<  只允许输入实数，包括.
        LXDRestrictTypeOnlyCharacter = 3,  ///<  只允许非中文输入
    };
    
在文本输入的时候会有两次回调，一次是代理的`replace`的替换文本方法，另一个需要我们手动添加的`EditingChanged`编辑改变事件。前者在中文联想输入的时候无法准确获取文本内容，而当确认好输入的文本之后才会调用后面一个事件，因此回调后一个事件才能准确的筛选文本。下面的代码会筛选掉文本中所有的非数字：

    - (void)viewDidLoad
    {
        [textField addTarget: self action: @selector(textDidChanged:) forControlEvents: UIControlEventEditingChanged];
    }

    - (void)textDidChanged: (UITextField *)textField
    {
        NSMutableString * modifyText = textField.text.mutableCopy;
        for (NSInteger idx = 0; idx < modifyText.length; idx++) {
            NSString * subString = [modifyText substringWithRange: NSMakeRange(idx, 1)];
            // 使用正则表达式筛选
            NSString * matchExp = @"^\\d$";
            NSPredicate * predicate = [NSPredicate predicateWithFormat: @"SELF MATCHES %@", matchExp];
            if ([predicate evaluateWithObject: subString]) {
                idx++;
            } else {
                [modifyString deleteCharactersInRange: NSMakeRange(idx, 1)];
            }
        }
    }
<span><img src="/images/UITextField的那点事/1.gif" width="600"></span>

限制扩展
----
如果说我们每次需要限制输入的时候都加上这么一段代码也是有够糟的，那么如何将这个功能给封装出来并且实现自定义的限制扩展呢？笔者通过工厂来完成这一个功能，每一种文本的限制对应一个单独的类。抽象提取出一个父类，只提供一个文本变化的实现接口和一个限制最长输入的`NSUInteger`整型属性：


    pragma mark - h文件
    @interface LXDTextRestrict : NSObject

    @property (nonatomic, assign) NSUInteger maxLength;
    @property (nonatomic, readonly) LXDRestrictType restrictType;

    // 工厂
    + (instancetype)textRestrictWithRestrictType: (LXDRestrictType)restrictType;
    // 子类实现来限制文本内容
    - (void)textDidChanged: (UITextField *)textField;

    @end


    pragma mark - 继承关系
    @interface LXDTextRestrict ()

    @property (nonatomic, readwrite) LXDRestrictType restrictType;

    @end

    @interface LXDNumberTextRestrict : LXDTextRestrict
    @end

    @interface LXDDecimalTextRestrict : LXDTextRestrict
    @end

    @interface LXDCharacterTextRestrict : LXDTextRestrict
    @end

    pragma mark - 父类实现
    @implementation LXDTextRestrict

    + (instancetype)textRestrictWithRestrictType: (LXDRestrictType)restrictType
    {
        LXDTextRestrict * textRestrict;
        switch (restrictType) {
            case LXDRestrictTypeOnlyNumber:
                textRestrict = [[LXDNumberTextRestrict alloc] init];
                break;
            
            case LXDRestrictTypeOnlyDecimal:
                textRestrict = [[LXDDecimalTextRestrict alloc] init];
                break;
            
            case LXDRestrictTypeOnlyCharacter:
                textRestrict = [[LXDCharacterTextRestrict alloc] init];
                break;
            
            default:
                break;
        }
        textRestrict.maxLength = NSUIntegerMax;
        textRestrict.restrictType = restrictType;
        return textRestrict;
    }

    - (void)textDidChanged: (UITextField *)textField
    {
    
    }

    @end
    
由于子类在筛选的过程中都存在遍历字符串以及正则表达式验证的流程，把这一部分代码逻辑给封装起来。根据`EOC`的原则优先使用`static inline`的内联函数而非宏定义：

    typedef BOOL(^LXDStringFilter)(NSString * aString);
    static inline NSString * kFilterString(NSString * handleString, LXDStringFilter subStringFilter)
    {
        NSMutableString * modifyString = handleString.mutableCopy;
        for (NSInteger idx = 0; idx < modifyString.length;) {
            NSString * subString = [modifyString substringWithRange: NSMakeRange(idx, 1)];
            if (subStringFilter(subString)) {
                idx++;
            } else {
                [modifyString deleteCharactersInRange: NSMakeRange(idx, 1)];
            }
        }
        return modifyString;
    }

    static inline BOOL kMatchStringFormat(NSString * aString, NSString * matchFormat)
    {
        NSPredicate * predicate = [NSPredicate predicateWithFormat: @"SELF MATCHES %@", matchFormat];
        return [predicate evaluateWithObject: aString];
    }


    pragma mark - 子类实现
    @implementation LXDNumberTextRestrict

    - (void)textDidChanged: (UITextField *)textField
    {
        textField.text = kFilterString(textField.text, ^BOOL(NSString *aString) {
            return kMatchStringFormat(aString, @"^\\d$");
        });
    }

    @end

    @implementation LXDDecimalTextRestrict

    - (void)textDidChanged: (UITextField *)textField
    {
        textField.text = kFilterString(textField.text, ^BOOL(NSString *aString) {
            return kMatchStringFormat(aString, @"^[0-9.]$");
        });
    }

    @end

    @implementation LXDCharacterTextRestrict

    - (void)textDidChanged: (UITextField *)textField
    {
        textField.text = kFilterString(textField.text, ^BOOL(NSString *aString) {
            return kMatchStringFormat(aString, @"^[^[\\u4e00-\\u9fa5]]$");
        });
    }

    @end
    
有了文本限制的类，那么接下来我们需要新建一个`UITextField`的分类来添加输入限制的功能，主要新增三个属性：

    @interface UITextField (LXDRestrict)

    /// 设置后生效
    @property (nonatomic, assign) LXDRestrictType restrictType;
    /// 文本最长长度
    @property (nonatomic, assign) NSUInteger maxTextLength;
    /// 设置自定义的文本限制
    @property (nonatomic, strong) LXDTextRestrict * textRestrict;

    @end
    
由于这些属性是`category`中添加的，我们需要手动生成`getter`和`setter`方法，这里使用`objc_associate`的动态绑定机制来实现。其中核心的方法实现如下：

    - (void)setRestrictType: (LXDRestrictType)restrictType
    {
        objc_setAssociatedObject(self, LXDRestrictTypeKey, @(restrictType), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        self.textRestrict = [LXDTextRestrict textRestrictWithRestrictType: restrictType];
    }

    - (void)setTextRestrict: (LXDTextRestrict *)textRestrict
    {
        if (self.textRestrict) {
            [self removeTarget: self.text action: @selector(textDidChanged:) forControlEvents: UIControlEventEditingChanged];
        }
        textRestrict.maxLength = self.maxTextLength;
        [self addTarget: textRestrict action: @selector(textDidChanged:) forControlEvents: UIControlEventEditingChanged];
        objc_setAssociatedObject(self, LXDTextRestrictKey, textRestrict, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
完成这些工作之后，只需要一句代码就可以完成对`UITextField`的输入限制：

    self.textField.restrictType = LXDRestrictTypeOnlyDecimal;
    
<span><img src="/images/UITextField的那点事/2.gif" width="600"></span>

自定义的限制
----
假如现在文本框限制只允许输入`emoji`表情，上面三种枚举都不存在我们的需求，这时候自定义一个子类来实现这个需求。

    @interface LXDEmojiTextRestrict : LXDTextRestrict

    @end

    @implementation LXDEmojiTextRestrict

    - (void)textDidChanged: (UITextField *)textField
    {
        NSMutableString * modifyString = textField.text.mutableCopy;
        for (NSInteger idx = 0; idx < modifyString.length;) {
            NSString * subString = [modifyString substringWithRange: NSMakeRange(idx, 1)];
            NSString * emojiExp = @"^[\\ud83c\\udc00-\\ud83c\\udfff]|[\\ud83d\\udc00-\\ud83d\\udfff]|[\\u2600-\\u27ff]$";
            NSPredicate * predicate = [NSPredicate predicateWithFormat: @"SELF MATCHES %@", emojiExp];
            if ([predicate evaluateWithObject: subString]) {
                idx++;
            } else {
                [modifyString deleteCharactersInRange: NSMakeRange(idx, 1)];
            }
        }
        textField.text = modifyString;
    }

    @end
    
代码中的`emoji`的正则表达式还不全，因此在实践中很多的`emoji`点击会被筛选掉。效果如下：
<span><img src="/images/UITextField的那点事/3.gif" width="600"></span>

键盘遮盖
----
另一个让人头疼的问题就是输入框被键盘遮挡。这里通过在`category`中添加键盘相关通知来完成移动整个`window`。其中通过下面这个方法获取输入框在`keyWindow`中的相对坐标：

    - (CGPoint)convertPoint:(CGPoint)point toView:(nullable UIView *)view
    
我们给输入框提供一个设置自动适应的接口：

    @interface UITextField (LXDAdjust)

    /// 自动适应
    - (void)setAutoAdjust: (BOOL)autoAdjust;

    @end

    @implementation UITextField (LXDAdjust)

    - (void)setAutoAdjust: (BOOL)autoAdjust
    {
        if (autoAdjust) {
            [[NSNotificationCenter defaultCenter] addObserver: self selector: @selector(keyboardWillShow:) name: UIKeyboardWillShowNotification object: nil];
            [[NSNotificationCenter defaultCenter] addObserver: self selector: @selector(keyboardWillHide:) name: UIKeyboardWillHideNotification object: nil];
        } else {
            [[NSNotificationCenter defaultCenter] removeObserver: self];
        }
    }

    - (void)keyboardWillShow: (NSNotification *)notification
    {
        if (self.isFirstResponder) {
            CGPoint relativePoint = [self convertPoint: CGPointZero toView: [UIApplication sharedApplication].keyWindow];
        
            CGFloat keyboardHeight = [notification.userInfo[UIKeyboardFrameBeginUserInfoKey] CGRectValue].size.height;
            CGFloat actualHeight = CGRectGetHeight(self.frame) + relativePoint.y + keyboardHeight;
            CGFloat overstep = actualHeight - CGRectGetHeight([UIScreen mainScreen].bounds) + 5;
            if (overstep > 0) {
                CGFloat duration = [notification.userInfo[UIKeyboardAnimationDurationUserInfoKey] doubleValue];
                CGRect frame = [UIScreen mainScreen].bounds;
                frame.origin.y -= overstep;
                [UIView animateWithDuration: duration delay: 0 options: UIViewAnimationOptionCurveLinear animations: ^{
                    [UIApplication sharedApplication].keyWindow.frame = frame;
                } completion: nil];
            }
        }
    }

    - (void)keyboardWillHide: (NSNotification *)notification
    {
        if (self.isFirstResponder) {
            CGFloat duration = [notification.userInfo[UIKeyboardAnimationDurationUserInfoKey] doubleValue];
            CGRect frame = [UIScreen mainScreen].bounds;
            [UIView animateWithDuration: duration delay: 0 options: UIViewAnimationOptionCurveLinear animations: ^{
                [UIApplication sharedApplication].keyWindow.frame = frame;
            } completion: nil];
        }
    }

    - (void)dealloc
    {
        [[NSNotificationCenter defaultCenter] removeObserver: self];
    }

    @end
    
<span><img src="/images/UITextField的那点事/4.gif" width="600"></span>
如果项目中存在自定义的`UITextField`子类，那么上面代码中的`dealloc`你应该使用`method_swillzing`来实现释放通知的作用


尾语
----
其实大多数时候，实现某些小细节功能只是很简单的一些代码，但是需要我们去了解事件响应的整套逻辑来更好的完成它。另外，昨天给微信`小程序`刷屏了，我想对各位iOS开发者说与其当心自己的饭碗是不是能保住，不如干好自己的活，顺带学点js适应一下潮流才是王道。[本文demo](https://github.com/JustKeepRunning/LXDTextFieldAdjust)

转载请注明本文地址及作者


