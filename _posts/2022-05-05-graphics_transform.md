---
title: 图形学笔记-变换
date: 2022-05-05 08:00
tags:
- Graphics
---

## 开篇
> 计算机图形学(Computer Graphics)是一种使用数学算法将二维或三维图形转换为计算机显示器的栅格形式的科学

本文为学习图形学过程中，使用`iOS`平台开发语言**模拟图像渲染和变化**的笔记

## 系列往期
[图形学笔记-渲染](http://sindrilin.com/2022/05/02/graphics_pixels_render.html)

## 欧拉角
欧拉角表示三维空间中可以任意旋转的`3`个值，分别是俯仰角(`Pitch`)、偏航角(`Yaw`)和滚转角(`Roll`)，可以认为是物体基于自身坐标系分别沿着`X`轴、`Y`轴和`Z`轴做旋转的变换

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acc6a4fa53bc403685f9da633d0dd0d8~tplv-k3u1fbpfcp-zoom-1.image)

## 变换矩阵
向量`AB`旋转`β`角度到了`AB'`的位置，可以得到

    B'.x = |AB| * cos(α+β) = |AB| * (cosα*cosβ - sinα*sinβ) = B.x * cosβ - B.y * sinβ
    B'.y = |AB| * sin(α+β) = |AB| * (cosα*sinβ + sinα*cosβ) = B.x * sinβ + B.y * cosβ

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ab78d1b6bc043af976848ef45f1b01a~tplv-k3u1fbpfcp-watermark.image?)

可以得到右手坐标系下的旋转矩阵：

    #define DEGRESS_TO_RADIANS($degress)  (($degress) / (180.0 / M_PI))
    
    #define MATRIX_SIZE 4
    typedef float Matrix4[MATRIX_SIZE][MATRIX_SIZE];
    
    void rightHandTransformMatrix(Matrix4 matrix, RotateAxis axis, float angle) {
        switch (axis) {
            case X:
                matrix[1][1] = cos(DEGRESS_TO_RADIANS(angle));
                matrix[1][2] = -sin(DEGRESS_TO_RADIANS(angle));
                matrix[2][1] = sin(DEGRESS_TO_RADIANS(angle));
                matrix[2][2] = cos(DEGRESS_TO_RADIANS(angle));
                break;
            case Y:
                matrix[0][0] = cos(DEGRESS_TO_RADIANS(angle));
                matrix[0][2] = sin(DEGRESS_TO_RADIANS(angle));
                matrix[2][0] = -sin(DEGRESS_TO_RADIANS(angle));
                matrix[2][2] = cos(DEGRESS_TO_RADIANS(angle));
                break;
            case Z:
                matrix[0][0] = cos(DEGRESS_TO_RADIANS(angle));
                matrix[0][1] = -sin(DEGRESS_TO_RADIANS(angle));
                matrix[1][0] = sin(DEGRESS_TO_RADIANS(angle));
                matrix[1][1] = cos(DEGRESS_TO_RADIANS(angle));
                break;
        }
    }
    
由于`iOS`设备采用的是左上角坐标原点，`y`轴向下增加，因此适用的是左手坐标系，将变换矩阵的`sin`取反即完成适配

## 深度
三维坐标系中，通常`z`轴代表了物体的距离视角的距离信息，`z`轴数值越大，表示越接近显示屏幕。在下图有过`ABC`三个点形成的平面，和过`DEF`三个点形成的平面，由于`ABC`平面的`z`值大于`DEF`平面，由于`ABC`平面非透明，所以`DEF`不会被显示

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64287887d2124128a0a4beebe0027f26~tplv-k3u1fbpfcp-watermark.image?)

当`ABC`平面沿着`x`轴旋转后，直接的观感就是整个平面的高度坍缩。假如把坐标轴的刻度当做屏幕渲染点(`dp`)，就可以当做旋转发生后，单个`dp`容纳了比原图更多的像素，最终`dp`渲染的颜色由深度(`z`轴数值)更大的像素表示

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c967184947954380897b24e1cb1c50e7~tplv-k3u1fbpfcp-watermark.image?)

## 变换实现
针对变换实现，加入了`Position`的三维坐标模型，和`SDLTransform`用来做矩阵变换的工具类

    typedef struct Position {
        float x;
        float y;
        float z;
    } Position;
    
    extern Position MakePosition(float x, float y, float z);
    
    @interface SDLTransform : NSObject
    
    /// 坐标轴的旋转角度
    @property (nonatomic, assign) CGFloat pitch;
    @property (nonatomic, assign) CGFloat yaw;
    @property (nonatomic, assign) CGFloat roll;
    
    /// 平移
    @property (nonatomic, assign) Position translation;
    
    @end
    
### 变换顺序
对于要渲染的内容分别先做平移、然后旋转，或是先旋转、再旋转的两种处理最终的计算结果完全不同，借用[GAMES 101](https://www.bilibili.com/video/BV1X7411F744?p=3)课程的截图说明这两种先后顺序最终的变换结果

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71d5c504b3aa4df09ac39b3ced548a22~tplv-k3u1fbpfcp-watermark.image?)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9df3787fa8f4b699bf7b79158fd0cf6~tplv-k3u1fbpfcp-watermark.image?)

因此在做变换之前，必须先将渲染图像修正到正确的中心坐标再做处理，才能得到预期的结果：

    @implementation SDLTransform
    
    - (Position)transformCoordinate:(Position)coordinate origin:(Position)origin {
        coordinate = MinusPosition(coordinate, origin);
        if (self.pitch != NONE_PITCH) {
            Matrix4 mat4;
            rotateMatrix(mat4, X, self.pitch);
            [self transformPosition:coordinate mat:mat4];
        }
        if (self.yaw != NONE_YAW) {
            // Y轴旋转
        }
        // Z轴旋转 & 平移
        return AddPosition(coordinate, origin);
    }
    
    @end
    
### 点渲染
由于旋转后一个屏幕渲染点(`dp`)可以容纳多个像素，如果存在不透明的像素，那么所有`z`轴数值低于这个像素的其他像素都可以不做渲染

    @implementation SDLRenderPoint
    
    - (void)appendPixel:(RGBAColor)color depth:(float)depth {
        NSInteger depthIndex = [self insertIndexOf:depth];
        if (color.alpha == NONE_ALPHA) {
            [self removeAllPixelsAfterDepth:depth];
        }
        [self insertPixel:(RGBAColor)color atIndex:depthIndex];
    }
    
    @end
    
最终是单个渲染点的颜色混合计算

    @implementation SDLRenderPoint
    
    - (RGBAColor)renderColor {
        if ([self isEmpty]) {
            return clearColor();
        }
        return [self mixedPixelsWithOptions:SDLMixedReverse:usingBlock:^RGBAColor(RGBAColor current, RGBAColor previous) {
            return [self mixedForegroundColor:current backgroundColor:previous];
        }];
    }
    
    @end
    
### 效果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4179fa120614b7eb90286e6d5e34067~tplv-k3u1fbpfcp-watermark.image?)