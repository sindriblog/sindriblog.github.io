---
title: 图形学笔记-光照
date: 2022-05-12 08:00
tags:
- Graphics
---

## 开篇
> 计算机图形学(Computer Graphics)是一种使用数学算法将二维或三维图形转换为计算机显示器的栅格形式的科学

本文为学习图形学过程中，使用`iOS`平台开发语言**模拟图像渲染和变化**的笔记

## 系列往期
[图形学笔记-渲染](http://sindrilin.com/2022/05/02/graphics_pixels_render.html)

[图形学笔记-变换](http://sindrilin.com/2022/05/05/graphics_transform.html)

## 物体显色
自然光是由多种不同波长的光组成的，通过凸透镜等工具折射反射后，会呈现出不同波长的光，表现为不同的颜色，例如彩虹形成的七种颜色就是自然光折射呈七种不同波长的光线进入肉眼后看到的。由于不同物质会吸收不同波长的光，反射其不能吸收的光，反射的可见光就被会被当做该物体的颜色。打个比方，使用`RGB`颜色系统，自然光可以认为是白色，其表示为

    lightColor = RGB(1, 1, 1)
    
一个看起来是红色的物体，自然光在入射到物体上时，发生了光线的吸收和反射，将这一过程的用代码可以描述成

    @implementation RedItem
    
    - (RGBColor)absorbLight:(RGBColor)lightColor {
        RGBColor itemColor = lightColor;
        itemColor.green = 0;
        itemColor.blue = 0;
        return itemColor;
    }
    
    @end
    
描述物体吸收自然光的过程并不直观，更好的方式是去计算物体反射的部分，用代码描述就可以表示为

    @implementation RedItem
    
    - (instancetype)init {
        if (self = [super init]) {
            self.itemColor = RGBColor(1, 0, 0);
        }
        return self;
    }
    
    - (RGBColor)absorbLight:(RGBColor)lightColor {
        return self.itemColor * lightColor;
    }
    
    @end

## 光照意义
在不考虑自然光的情况下，日常中使用的光线是随距离衰减的，比如手电筒、手机闪光灯等，光照的意义之一在于帮助人类估算距离，距离更远的物体必然色彩更不明显；反之如果没有合适的光照，会有错误的视觉效果

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ade28f34d8e46a1b7833c51fa12f9a5~tplv-k3u1fbpfcp-watermark.image?)

以上一篇《变换》做距离，如下图，垂直方向上为沿着`x`轴每次向上旋转`30°`，水平方向上为沿着`y`轴每次向左旋转`30°`。在不考虑透视的情况下，图像和向下旋转、向右旋转相同角度也是一样的，甚至还可以被看做只是`x`轴和`y`轴上的缩放。因此如果能模拟

## 光线计算
本文谈及的`点光源`和`球光源`是基于模拟光照计算的描述，并不指代其原含义（ps：命名请勿较真）

### 点光源
点光源是一种光线强度只和入射角度有关，当入射光线与物体面垂直（`x`与`y`轴坐标相等时）时，光线强度最高

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98f47126b082489fb0abdb64ae44f90e~tplv-k3u1fbpfcp-watermark.image?)

由图可以看出点光源的计算非常简单，即求点光源到入射点夹角的余弦`cosθ`即可，在给定了点光源坐标点`lightPosition`和入射点坐标`pixelPosition`的情况下，入射光强度计算为

    RGBColor calculateColor(RGBColor color, Position lightPosition, Position pixelPosition) {
        Position referencePosition = MakePosition(lightPosition.x, lightPosition.y, pixelPosition.z);
        float referenceDistance = PositionDistance(referencePosition, lightPosition);
        float pixelDistance = PositionDistance(pixelPosition, lightPosition);
        return color * (referenceDistance / pixelDistance);
    }

通过添加点光源，就能明显感受到旋转感


![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/540c742be73747cbb94bad606b20584c~tplv-k3u1fbpfcp-watermark.image?)

### 球光源
将光源看做是一个球体，光线不断的向外扩散，光线从光源中心出发后，在不同时刻形成了以光源为中心的一个个球体，球体的表面积`S=4πr²`看做是形成球体的光线强度的总和

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8bad0d30aae473c97690ecf5aa42ffc~tplv-k3u1fbpfcp-watermark.image?)

以光线到光源中心的距离生成半径为`R`的球体，任意扩散两个时间点`t1`和`t2`，分别形成的光线球体的强度和`S1`和`S2`是相等的。基于这个理论，计算像素点的光照强度时，可以换算成计算像素点到光源的距离形成的`R`的光球表面单个点的能量。为了便于计算，我们需要引入一些变量

- 标准光强距离`Intensity`：表示球体表面单个点光线强度为`1`时的`R`值，光球强度总和为`4πI²`

基于标准值，可以得到任意像素点受到的光照强度计算

    @implementation SDLLightSource
    
    - (void)setIntensity:(CGFloat)intensity {
        _intensity = intensity;
        self.totalPower = 4 * M_PI * intensity * intensity;
    }

    - (CGFloat)lightIntensityAtPosition:(Position)position {
        float distance = PositionDistance(self.position, poisition);
        return self.totalPower / (4 * M_PI * distance * distance);
    }

    @end
    
通过计算公式可以看出球光源单点的光线强度随距离平方衰减，比点光源计算量也要大，但换来了更真实的光照效果

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1fa13ddb76348dba87cfcabe5d7b794~tplv-k3u1fbpfcp-watermark.image?)

### 光照颜色
有时候光源不一定标准的`RGB(1, 1, 1)`的白色，但无论光源颜色是否为白色，都不影响效果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1161bc3580b0478caf2c68ccb9ffb2f6~tplv-k3u1fbpfcp-watermark.image?)

## 半透明照射
上面采用的例子最顶层的图像都是非透明的，现在试着把大黄脸改成`60%`的透明度

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/290ce7dc0cd0497b88fe64d7c12b5254~tplv-k3u1fbpfcp-watermark.image?)

看起来还可以，但实际上是不对的，上面两个图像分别和光源做了单独的光照计算后再进行像素混合计算，这个过程忽略了一个问题，如下图所示，光线穿过半透明的大黄脸后，必然会被吸收掉一部分，那么穿过大黄脸的光线颜色是什么？

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53c708fe9f354b8c9c812488e8801d97~tplv-k3u1fbpfcp-watermark.image?)

读书的时候有个小游戏，给一张半透明的红色纸片，和一张黄色的纸张，打开手电筒让光线穿过红色纸片照射到黄色纸片上，最终会在黄色纸片上出现橙色

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/467b613a46b649c5a80c6b832f0e2f29~tplv-k3u1fbpfcp-watermark.image?)

由此可以假设一些规则

1. 光线穿过半透明物体时，物体的透明度为`alpha`，穿透的光线比例为`1-alpha`
2. 穿透色必定为物体不可吸收的颜色（物体反射色）

假设`A`像素点是半透明像素点，`B`是在`A`像素点后方的像素点，可以得到

    B = A * lightColor * (1 - A.alpha)

### 效果
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f783be785824088a8b9678f0d7b36e3~tplv-k3u1fbpfcp-watermark.image?)