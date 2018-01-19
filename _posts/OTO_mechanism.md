---
title: 一对一机制
date: 2018-01-06 08:00:00
categories:
- Note
tags:
- Note
- Tips
---

一对一`one to one`并不是某种业务上的具体需求，它仅仅是一种数据存储上的设计。`one to one`要求一个变量能且最多只能匹配一个变量，这听着很绕口，举个常见的结构例子就是`key-value`匹配。不同的编程语言提供了高层级的`key-value`存储结构，例如`dictionary`字典，被这些高层级的存储结构宠坏了的我们不知是否思考过这么一个问题：

> `dictionary`是怎么通过`key`来存储或者移除匹配的`value`的

应用程序的数据存储只存在`连续存储`和`非连续存储`这两种方式，并且`非连续存储`需要使用额外的内存来存储下一个数据的信息，因此`非连续存储`的内存开销是大于`连续存储`方式的。在`dictionary`的存储结构中，`key`和`value`可以完全不同且没有丝毫的关联性，这是否意味着字典会采用`非连续存储`的结构设计并且使用了大量的额外内存呢？又或者存在某种设计，使得即便是完全不同的数据，也能实现连续的存储呢？

## 从排序说起
如果你已经了解过`桶排序`或者`哈希表`，那么可以跳过本节继续阅读。

### 桶排序
在代码设计中，`空间`和`时间`是两个非常重要的概念，前者表示代码运行过程中，需要占用的内存大小。后者表示运行这段代码，`CPU`需要处理多少个时钟周期。实现同样一个功能需要的`空间`和`时间`并不是固定的，通常来说，前者相比后者要廉价的多。因此在实现需求的时候，我们可以适当的放弃一些`空间`来换取更快的执行效率。`桶排序`是一种典型的`空间换时间`算法，它采用一种十分取巧的方式，让排序时间可以达到`O(N)`的水平。

> 创建`N`个空桶，`N`为排序数组中最大值加一。然后遍历排序数组，以元素值为下标，将其放入对应的桶中

出于读者大多数为`iOSer`的考虑，我将使用`OC`代码来描述桶排序。首先存在一个需要排序的无序正整数数组，我们遍历这个数组，然后创建一堆空桶：

    NSUInteger bucketCount = 0;
    NSArray<NSNumber *> *nums = @[@(10), @(8), @(3), @(9), @(3), @(1), @(4), @(6), @(7), @(5)];
    for (NSNumber *num in nums) {
        if (bucketCount < num.unsignedIntegerValue + 1) {
            bucketCount = num.unsignedIntegerValue + 1;
        }
    }
    
    NSMutableArray<NSNumber *> *buckets = @[].mutableCopy;
    for (NSUInteger idx = 0; idx < bucketCount; idx++) {
        [buckets addObject: @0];
    }
    
因为`桶排序`的思想是：数组中每一个元素都能成为桶列表的下标，所以桶列表的长度必须为`最大值加一`才能避免数组访问越界。桶列表被初始化后，每一个桶的值都是`0`，表示该桶的数量为0个。然后对原数组进行一次遍历，将元素放到对应的桶中（桶内值`+1`）：

    for (NSNumber *num in nums) {
        NSUInteger count = buckets[num.unsignedIntegerValue].unsignedIntegerValue;
        buckets[num.unsignedIntegerValue] = @(count + 1);
    }

    NSMutableArray<NSNumber *> *sortNums = @[].mutableCopy;
    for (NSUInteger idx = 0; idx < buckets.count; idx++) {
        NSUInteger count = buckets[idx].unsignedIntegerValue;
        while (count--) {
            [sortNums addObject: @(idx)];
        }
    }
    
![](http://p0zs066q3.bkt.clouddn.com/2018012101.jpg)
    
在两次遍历之后，数组值已经将元素转换成下标顺序的存放在桶列表中，最后遍历所有桶，如果桶不为空，那么取下标，遍历后排序完成。

### 哈希排序
`桶排序`效率非常的高，但是存在一个致命的问题：内存占用过大。当需要处理的数据足够大时，计算机根本无法分配足够多的桶来完成排序。基于这一问题，`哈希排序`将较大的数值映射成较小的数值，成倍的减少了需要使用到的桶的数量。由于`较大数值->较小数值`的过程中，会导致数据精确的丢失，同一个桶可能存在多个排序数据，因此每一个桶存储的应该是一个数组：

    NSUInteger hashCode(NSNumber *num) {
        return num.unsignedIntegerValue / 10;
    }

    NSUInteger bucketCount = 0;
    NSArray<NSNumber *> *nums = @[@(101), @(89), @(32), @(95), @(33), @(11), @(44), @(62), @(71), @(99)];
    for (NSNumber *num in nums) {
        if (bucketCount < (hashCode(num) + 1)) {
            bucketCount = (hashCode(num) + 1);
        }
    }
    
    NSMutableArray<NSNumber *> *buckets = @[].mutableCopy;
    for (NSUInteger idx = 0; idx < bucketCount; idx++) {
        [buckets addObject: @[].mutableCopy];
    }

    for (NSNumber *num in nums) {
        NSMutableArray *bucket = buckets[num.unsignedIntegerValue / 10];
        [bucket addObject: num];
    }

由于同一个桶中可能存在多个待排序数据，因此在最后完成排序操作中，还需要进行额外的排序。下面略去对桶中数据重新排序的代码：

    NSMutableArray<NSNumber *> *sortNums = @[].mutableCopy;
    for (NSUInteger idx = 0; idx < buckets.count; idx++) {
        NSMutableArray *bucket = buckets[idx];
        [sortNums addObjectsFromArray: [bucket sortedArrayUsingSelector: @selector(compare:)]];
    }

![](http://p0zs066q3.bkt.clouddn.com/2018012102.jpg)

相较于`桶排序`，`哈希排序`在`空间`和`时间`上努力寻找一个平衡点，尽可能的保证效率。但实际上当处理数据足够大量时，`哈希排序`相较于其他的排序方式，并没有体现足够的优势，因此在`排序`上，这并不是一种很好的选择。但缩小数据范围的思路，却体现了不同数据连续存储的可能性。

## 字典



