---
title: 图形学笔记-渲染
date: 2022-05-02 08:00
tags:
- Graphics
---

## 开篇
> 计算机图形学(Computer Graphics)是一种使用数学算法将二维或三维图形转换为计算机显示器的栅格形式的科学

本文为学习图形学过程中，使用`iOS`平台开发语言**模拟图像渲染和变化**的笔记

## 像素渲染
将像素展示到屏幕上需要借助`CoreGraphics`的接口，假设需要渲染`width * height`的图像，需要先分配色彩空间为`ARGB`的位图

    #define BYTES_PER_RGB 4
    
    typedef struct SDLBitmapContext {
        CGSize size;
        uint32_t *buffer;
        CGContextRef cgContext;
        CGColorSpaceRef colorSpace;
    } SDLBitmapContext;
    
    SDLBitmapContext sdlCreateArgbBitmapContext(size_t width, size_t height) {
        SDLBitmapContext context = identityBitmapContext();
        size_t bytesPerRow = width * BYTES_PER_RGB;
        size_t bytesCount = bytesPerRow * height;
        context.size = CGSizeMake(width, height);
        context.colorSpace = CGColorSpaceCreateDeviceRGB();
        if (context.colorSpace == NULL) {
            return identityBitmapContext();
        }
        
        context.buffer = (uint32_t *)malloc((unsigned long)bytesCount);
        if (context.buffer == NULL) {
            sdlFreeBitmapContext(&context);
            return identityBitmapContext();
        }
        memset(context.buffer, 0x0, bytesCount);
        
        context.cgContext = CGBitmapContextCreate(context.buffer, width, height, 8, bytesPerRow, context.colorSpace, kCGImageAlphaPremultipliedFirst);
        if (context.cgContext == NULL) {
            sdlFreeBitmapContext(&context);
            return identityBitmapContext();
        }
        return context;
    }

在分配完位图后，遍历位图的内存将像素数据写入`buffer`中

    enum {
        ALPHA = 0,
        RED = 1,
        GREEN = 2,
        BLUE = 3,
    };
    
    void sdlForeachBitmapContext(SDLBitmapContext context, SDLBitmapPixelBody body) {
        uint32_t *imageBuffer = CGBitmapContextGetData(context.cgContext);
        if (imageBuffer == NULL) {
            return;
        }
        for (size_t row = 0; row < context.size.width; row++) {
            for (size_t column = 0; column < context.size.height; column++) {
                uint8_t *pixelPtr = (uint8_t *)imageBuffer;
                body(pixelPtr, row, column);
            }
        }
    }
    
    sdlForeachBitmapContext(context, ^(uint8_t *pixelPtr, size_t row, size_t column) {
        RGBAColor color = randomColor();
        pixelPtr[ALPHA] = color.alpha;
        pixelPtr[RED] = color.red;
        pixelPtr[GREEN] = color.green;
        pixelPtr[BLUE] = color.blue;
    });

最后从位图上下文取出图片即可展示

    UIImage *sdlGenerateImage(SDLBitmapContext context) {
        if (!isBitmapContextValid(context)) {
            return nil;
        }
        CGImageRef imageRef = CGBitmapContextCreateImage(context.cgContext);
        UIImage *image = [UIImage imageWithCGImage:imageRef];
        CGImageRelease(imageRef);
        return image;
    }
    
    - (void)renderImage:(SDLBitmapContext context) {
        UIImage *image = sdlGenerateImage(context);
        self.renderImageView.image = image;
    }

随机像素渲染效果

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d5af3cb786ee42129af736244c5384a8~tplv-k3u1fbpfcp-watermark.image?)

## 图片渲染
图片的渲染和随机像素渲染的过程相似，同样需要生成`width * height`大小尺寸的位图，然后将图片写入到位图中，这次使用一个`PixelsStorage`模型来存储所有的像素信息

    @interface SDLPixelsStorage : NSObject
    
    /// 图片尺寸
    @property (nonatomic, readonly) CGSize imageSize;
    /// 像素存储buffer
    @property (nonatomic, readonly) RGBAColor *pixelColors;
    
    /// 生成指定大小的像素缓存区
    - (void)createImageWithSize:(CGSize)size;
    /// 遍历所有像素点
    - (void)enumeratePixelsWithVisitor:(SDLVisitor)visitor;
    /// 修改坐标点的像素值
    - (void)modifyColor:(RGBAColor)color atPoint:(CGPoint)point;
    
    @end
    
    - (SDLPixelsStorage *)readPixelsFromImage:(UIImage *)image {
        SDLBitmapContext context = sdlCreateArgbBitmapContext(image.size.width, image.size.height);
        SDLPixelsStorage *storage = [[SDLPixelsStorage alloc] init];
        [storage createImageWithSize:size];
        sdlDrawImageToBitmap(context, image.CGImage);
        sdlForeachBitmapPixel(context, ^(uint8_t *pixelPtr, size_t row, size_t column) {
            RGBAColor color = (RGBAColor){
                pixelPtr[RED],
                pixelPtr[GREEN],
                pixelPtr[BLUE],
                pixelPtr[ALPHA]
            };
            [storage modifyColor:color atPoint:CGPointMake(row, column)];
        });
        return storage;
    }
    
    - (void)viewDidLoad {
        [super viewDidLoad];
        SDLPixelsStorage *storage = [self readPixelsFromImage:[UIImage imageNamed:@"emoji"]];
        [[[SDLRenderCanvas alloc] initWithDisplayView:self.view pixelsStorage:storage] render];
    }

渲染效果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c243840700548e2adb074a5af26bfdc~tplv-k3u1fbpfcp-watermark.image?)

### 锯齿问题
由于`retina`屏幕的特性，在`3x`机型上单个坐标点共有`3 * 3 = 9`个像素，要达成最佳的图像渲染需要使用`3 * width * 3 * height`尺寸大小的位图，在此不做演示

### 为什么需要PixelsStorage
在后续文章中会模拟视图的旋转变换，需要对像素的每一个坐标点做矩阵变换得到新的坐标点，抽象出像素存储器的类可以很好的支持运算

## 混合图像
像素渲染时遵循一个规则：当像素`A`、`B`在同一坐标点且满足`A.z < B.z`时，假设`B`的透明度为`alpha`，坐标点的颜色计算公式为：

> A * (1 - B.alpha) + B * B.alpha

当`B.alpha`等于`1`时，`A`像素永远不会被表达，反之`B.alpha`不为1时，显示的颜色为像素的叠加。为了实现图像混合的效果，新增加一个`PixelsComposite`类实现效果

    @implementation SDLPixelsComposite
    
    - (SDLPixelsStorage *)compose {
        SDLPixelsStorage *composeStorage = [self normalizedStorage];
        SDLPixelsStorage *currentPixelsStorage = [[SDLPixelsStorage alloc] init];
        [currentPixelsStorage createImageWithSize:composeStorage.imageSize];
        
        // 遍历所有图像
        for (SDLPixelsComponent *component in self.pixelsComponents) {
            CGFloat widthScale = composeStorage.imageSize.width / component.size.width;
            CGFloat heightScale = composeStorage.imageSize.height / component.size.height;
            [component.pixels enumeratePixelsWithVisitor:^(RGBAColor color, CGPoint point) {
                 // 读取当前图像的色彩信息存储到currentPixelsStorage中
            }];
            
            // 混合图像像素
            CGFloat alpha = component.alpha;
            [composeStorage enumratePixelsWithVisitor:^(RGBAColor color, CGPoint point) {
                // A * (1 - B.alpha)
                RGBAColor ultimateColor = multipleColor(color, 1 - alpha);
                // B * B.alpha
                RGBAColor currentColor = multipleColor([currentPixelsStorage colorAtPoint:point]);
                ultimateColor = addColor(ultimateColor, currentColor);
                [composeStorage modifyColor:ultimateColor atPoint:point];
            }];
        }
        return composeStorage;
    }
    
    @end
    
    - (void)viewDidLoad {
        [super viewDidLoad];
        SDLPixelsComposite *composite = [[SDLPixelsComposite alloc] init];
        [composite appendPixels:[self readPixelsFromImage:@"container"] alpha:1];
        [composite appendPixels:[self readPixelsFromImage:@"emoji"] alpha:0.2];
        [[[SDLRenderCanvas alloc] initWithDisplayView:self.view pixelsStorage:[composite compose] render];
    }

渲染效果

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc96d0cee1654461978d8ad13ec5cd6c~tplv-k3u1fbpfcp-watermark.image?)

## 相关
[LearnOpenGL](https://learnopengl-cn.github.io)

[GAMES 101](https://www.bilibili.com/video/BV1X7411F744?spm_id_from=333.337.search-card.all.click)