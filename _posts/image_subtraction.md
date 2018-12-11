---
title: 质量监控-图片减包
date: 2018-12-11 08:00
tags:
- 质量监控
---

经过多个版本迭代，项目在`release`配置下的打包体积依旧轻松破百，应用体积过大导致的问题包括：

- 更长的构建时间，换个词就是`加班`
- `TEXT`段体积过大会导致审核失败
- 用户不愿意下载应用

通常来说，资源文件能在应用体积包中占据`1/3`或者更多的体积，相比起代码`（5kb/千行）`的平均占用来说，对图片进行减包是最直接高效的手段，对图片资源的处理方式包括四种：

1. 通过请求下载大图
2. 使用工具压缩图片
3. 查找删除重复图片
4. 查找复用相似图片

考虑到由于项目开发分工的问题，`方式1`需要推动落地，所以本文不讨论这种处理方式。其他三种都能通过编写脚本实现自动化处理

## 图片压缩
图片压缩分为`有损压缩`和`无损压缩`两类，`有损压缩`放弃了一部分图片的质量换取更高的压缩比。网上主流的压缩工具有`tinypng`、`pngquant`、`ImageAlpha`和`ImageOptim`等，分别采用了一种或者多种压缩技术完成图片压缩

### 为什么png能够无损压缩
由于`png`格式的灵活性，同一张图片可以使用多种方式进行表示，不同方式占用的大小不一样。一般的软件会采用效率更高的方式来表示图片，所以这种情况下`png`图片存在巨大的优化空间。通常来说，从`png`文件中能去除的数据包括：

- `iTXt`、`tEXt`和`zTXt`这些可以存储任意文本的数据区段
- `iCCP`数据区段存储的`profile`等等
- `photoshop`导出的`png`图片存在大量的额外信息

`png`图片有两种类型的数据块，一种是必不可缺的数据块称为`关键数据块`。另一种叫做`辅助数据块`，`png`文件格式规范指定的辅助数据块包括：

- 背景颜色数据块`bKGD`
- 基色和白色数据块`cHRM`
- 图像`γ`数据块`gAMA`
- 图像直方图数据块`hIST`
- 物理像素尺寸数据块`pHYs`
- 样本有效位数据块`sBIT`
- 文本信息数据块`tEXt`
- 图像最后修改时间数据块`tIME`
- 图像透明数据块`tRNS`
- 压缩文本数据块`zTXt`

其中`tEXt`和`zTXt`数据段中存在的数据包括：


| 关键字 |  |
| --- | --- |
| Title | 图像名称 |
| Author | 图像作者 |
| Description | 图像说明 |
| Copyright | 版权声明 |
| CreationTime | 原图创作时间 |
| Software | 创作图像使用的软件 |
| Disclaimer | 弃权 |
| Warning | 图像内容警告 |
| Source | 创作图像使用的设备 |
| Comment | 注释信息 |


由上可见，辅助数据块在`png`文件中可能占据了极大的篇幅，正是这些数据块构成了`png`的无损压缩条件

### tinypng
`tinypng`采用了一种称作`Quantization`的压缩技术，通过合并图片中相似的颜色，将`24bit`的图片文件压缩成`8bit`图片，同时去除图片中不必要的元数据，图片最高能达到`70%`以上的压缩率。截止文章完成之前，`tinypng`仅提供了线上压缩功能，暂未提供工具下载

### pngquant
根据官方介绍，`pngquant`将`24bit`以上的图片转换成`8bit`的保留透明度通道的压缩图片，压缩算法的压缩比非常显著，通常都能减少`70%`的大小。`pngquant`提供了命令行工具来完成解压任务：

    pngquant --quality=0-100 imagepath

命令行更多调试参数可以在[官网](https://pngquant.org)参阅

### ImageAlpha
`ImageAlpha`是一个`macOS`系统下的有损图片压缩工具，内置了`pngquant`、`pngnq-s9`等多个压缩工具，多数情况下通过将图片降至`8bit`来获取高压缩比。由于`ImageAlpha`的可视化界面无法批量处理图片，直接使用提供的命令工具可以实现批量压缩图片：

    for file in $(ls $1); do
        imagepath=$1"/"$file
        if [ -d imagepath ]
        then
            /// 路径为文件夹
        else
            if [[ $file == *.png ]]
            then
                beforeSize=`ls -l $imagepath | awk '{print $5}'`
                /Applications/ImageAlpha.app/Contents/MacOS/pngquant $imagepath
                afterSize=`ls -l ${imagepath/.png/-fs8.png} | awk '{print $5}'`
                
                if [[ $afterSize -lt $beforeSize]]
                then
                    mv ${imagepath/.png/-fs8.png} $imagepath
                fi
            fi
        fi
    done

使用`ImageAlpha`需要注意两点：

1. 压缩后的图片命名会自动添加`-fs8`后缀，需要使用`mv`命令实现替换
2. 有损压缩会修改`关键数据块`，可能导致压缩图片尺寸增大，需要过滤

在使用`有损压缩`时需要注意单张`png`图片是可以被多次压缩的，但这会导致图片的清晰度和色彩都受到影响，不建议对图片超过一次以上的`有损压缩`

### ImageOptim
`ImageOptim`是介绍的四种工具中唯一的`无损压缩`，它采用了包括`去除exif信息`、`重新排列像素存储方式`等手段实现图片的压缩。`无损`代表着一张图片被`ImageOptim`压缩后，后续无法再次进行压缩，同时它的压缩比往往比不上其他的`有损压缩`方案，但最大程度上保证了图片的原始清晰度和色彩

    for file in $(ls $1); do
        imagepath=$1"/"$file
        if [ -d imagepath ]
        then
            /// 路径为文件夹
        else
            if [[ $file == *.png ]]
            then
                /Applications/ImageOptim.app/Contents/MacOS/ImageOptim $imagepath
            fi
        fi
    done

`ImageOptim`同样存在可视化的工具并且支持批量压缩图片

### 多方案对比
考虑到`ImageAlpha`几乎都是使用`pngquant`作为压缩工具，因此只列出三种压缩工具的对比：


| 原始尺寸 | 压缩工具 | 压缩后尺寸 | 压缩比 |
| --- | --- | --- | --- |
| 319.5KB | tinypng | 120.5KB | 62% |
| 319.5KB | ImageAlpha-pngquant | 395KB | -24% |
| 319.5KB | ImageOptim | 252KB | 21% |

测试图片采用`qq`聊天截图生成的`png`，`tinypng`压缩率非常高，而`pngquant`的表现不尽人意

## 删除重复图片
通常来说，出现重复图片的原因包括`模块间需求开发没有打通`或是`缺少统一的图片命名规范`。通过图片`MD5`摘要是识别重复图片的最快方法，以`python`为例，匹配重复图片的代码如下：

    md5list = {}
    for file in files:
        if os.path.isdir(file.path):
            continue
            
        md5obj = hashlib.md5()
        fd = open(file.path, 'rb')
        while True:
            buff = fd.read(2048)
            if not buff:
                break
            md5obj.update(buff)
        fd.close()
        
        filemd5 = str(md5obj.hexdigest()).lower()
        if filemd5 in md5list:
            md5list[filemd5].add(file.path)
        else:
            md5list[filemd5] = set([file.path])
            
    for key in md5list:
        list = md5list[key]
        if len(list) > 1:
            print (list)
    
在遍历中以文件`MD5`字符串作为`key`，维护具备相同`MD5`的图片路径，最后遍历这个`map`查找存在一个以上路径的数组并且输出

## 寻找相似图片
相似图片在图片内容、色彩上都十分的接近，多数时间可以考虑复用这些图片，但相似图片的问题在于无法通过`MD5`直接匹配。为了确认两个图片是否相似，要使用简单的一个数学公式来帮忙查找：

> 方差。在概率论和统计学中，一个随机变量的方差描述的是它的离散程度，也就是该变量离其期望值的距离

举个例子，甲同学五次成绩分别是`65, 69, 81, 89, 96`，乙同学五次成绩是`82, 80, 77, 81, 80`，两个人平均成绩都是`80`，但是引入方差公式计算：

    甲： ((65-80)^2 + (69-80)^2 + (81-80)^2 + (89-80)^2 + (96-80)^2) / 5 = 136.8
    乙： ((82-80)^2 + (80-80)^2 + (77-80)^2 + (81-80)^2 + (80-80)^2) / 5 = 2.8
    
平均值相同的情况下，方差越大，说明数据偏离期望值的情况越严重。方差越接近的两个随机变量，他们的变化就越加趋同，获取方差代码如下：

    def getVariance(nums):
        variance = 0
        average = sum(nums) / len(nums)
        for num in nums:
            variance += (num - average) * (num - average) / len(nums)
        return variance
        
因此将图片划分成连串的一维数据，以此计算出图片的方差，通过方差匹配可以实现一个简单的图片相似度判断工具，实现前还要注意两点：

1. 图片`RGB`色彩值会导致方差的计算变得复杂，所以转成灰度图可以降低难度
2. 不同尺寸需要缩放到相同尺寸进行计算

最终将图片转换成一维数据列表的代码如下：
        
    def getAverageList(img):
        commonlength = 30
        img = cv2.resize(img, (commonlength, commonlength), interpolation=cv2.INTER_CUBIC)
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        
        res = []
        for idx in range(commonlength):
            average = sum(gray[idx]) / len(gray[idx])
            res.append(average)
            
将图片转成灰度图后，仍然可能存在`RGB`色值不同但灰度值相同的情况导致判断失准，可以考虑两种方案提高算法的检测准确率：

1. 在不修改以灰度值计算方差的方案下，构建以`列平均像素值`为单位的一维列表计算另一个方差，两个方差值一并做判断
2. 摒弃灰度值方差方案，每一行分别生成`R`、`G`、`B`三种色彩平均值的一维列表，计算出三个方差进行匹配检测

## 效果
经过两轮图片减包处理后，整个项目资源产生的减包量约有`20M`，其中通过文中的三种手段产生的减包量在`6.5M`左右，整体上来看产出还是比较可观的


