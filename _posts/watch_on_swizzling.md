---
title: 开发日记-警惕swizzling
date: 2018-09-14 12:00:00
tags: 开发日记
---

不知道什么时候开始，只要使用了`swizzling`都能被解读成是`AOP`开发，开发者张口嘴就是`runtime`，将其高高捧起，称之为`黑魔法`；以项目中各种`method_swizzling`为荣，却不知道这种做法破坏了代码的整体性，使关键逻辑支离破碎。本文基于[iOS界的毒瘤](https://www.valiantcat.cn/index.php/2017/11/03/53.html#menu_index_7)一文，从另外的角度谈谈为什么我们应当`警惕`

## 调用顺序性
`调用顺序性`是上述链接讲述的核心问题，它会破坏方法的原有执行顺序，导致流程错误。先从一段简单的代码聊起：

    @interface SLTestObject: NSObject
    
    @end
    
    @implementation SLTestObject
    /// empty
    @end
    
    SLTestObject *testObj = [[SLTestObject alloc] init];
    


