---
title: GIF 字节格式介绍
date: 2017-11-03 19:08:22
tags: 多媒体
cover: https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-9a836aaf2873ebc0.webp
---
![gif](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-9a836aaf2873ebc0.webp)


最近花了点时间用 C++ 写了一个 GIF 图片的解析程序，在这一过程中我找了许多中文相关的材料，但没有哪一篇是真正能够让读者完全理解 GIF 的文件格式和 LZW 在 GIF 中的应用（解析部分）。在查阅了一些官方文档后我算是顺利的将程序完成了，顺道我就把 GIF 文件的解析在这儿讲讲清除，方便大家学习。

下面这两个网页是我参考的比较权威的资料，大家也可以直接阅读。

> http://giflib.sourceforge.net/index.html
> https://www.w3.org/Graphics/GIF/spec-gif89a.txt
<!-- more -->
# GIF 文件格式

下图是我从 http://giflib.sourceforge.net/index.html 拿过来的图，从图上可以很清晰的看到 GIF 文件的内部组成。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-d2847ad5154c6744.webp)

上图中一个有十一块，其中有八块是 GIF 图像必备的：`Header`（头）、`Logical Screen Descriptor`（逻辑屏幕描述符）、`Image Descriptor`（图像描述符）、`Image Data`（图像数据流）、`Plain Text Extension`（文本扩展）、`Application Extension`（应用扩展）、`Comment Extension`（注释扩展）、`Trailer`（尾部标记）。

另外还有三个可选块：`Global Color Table`（全局颜色表）、`Graphic Control Extension`（图形控制扩展）、`Local Color Table`（本地颜色表）。

传统的 Gif 解码器正是通过上述的这些块来对 GIF 文件进行解析，下面我们就按照顺序来详细了解一下这些块的内部字节格式。

为了比较直观的了解这些内容，我还是从 http://giflib.sourceforge.net/index.html 拿了一个 GIF 图过来，如下图：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-434075727f059761.webp)

其放大后的效果图如下：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-9da5a168aebb3bf5.webp)

上图经过放大处理，其代表了一个 10*10 个像素的 GIF 图，其内部字节如下：

    47 49 46 38 39 61 
    0A 00 0A 00 91 00 00 
    FF FF FF FF 00 00 00 00 FF 00 00 00 
    21 F9 04 00 00 00 00 00 
    2C 00 00 00 00 0A 00 0A 00 00 
    02 16 8C 2D 99 87 2A 1C DC 33 A0 02 75 EC 95 FA A8 DE 60 8C 04 91 4C 01 00
    3B
    
## Header

GIF 文件由它的 Header 块最为其文件的入口，Header 一共包含六个字节，如下图：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-4309aa3f589fbaa8.webp)

其中前三个字节对应 ASCII 码中的 G、I、F 三个字符，后三个字节用于说明此 GIF 的版本号，目前的版本号有 87a 和 89a 两个。

对于上面的示例图来说，前六个字节分别是 47、49、46、38、39、61。 用 0xED 查看如下图：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-c558f44ea7b41241.webp)

## Logical Screen Descriptor

Logical Screen Descriptor（逻辑屏幕描述符）通常紧跟在 Header 后面，它的作用是告诉解码器 GIF 图像的分辨率，背景色和 Global Color Table 等信息。先看一下其字节的组成：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-49686736a5ecb836.webp)

按顺序看，

- `Canvas Width`：表示 GIF 图像的宽度，单位是像素。
- `Canvas Height`：表示 GIF 图像的高度，单位是像素。
- `Packed Field`：这是一个包装字段，内部的不同 bit（位）表示有不同的含义
    - 从左边数第一位表示 `Global Color Table Flag`，如果其为 1 ，则表示存在 Global Color Table。如果为 0，则没有 Global Color Table。
    - 从左边数第二、三、四位表示 `Color Resolution`，用于表示色彩分辨率，如果为 s，则 Global Color Table 的颜色数为 2^(s+1)个，如果这是 s = 1,则一共有 4 中颜色，即每个像素可以用 2位（二进制） 来表示。
    - 从左边数第五位表示 `Sort Flag`，它有两个值 0 或 1。如果为 0 则 Global Color Table 不进行排序，为 1 则表示 Global Color Table 按照降序排列，出现频率最多的颜色排在最前面。
    - 最右边三位表示 Global Color Table 的颜色数，如其值为 s，则全局列表颜色个数的计算公式为 2^(s+1)。如 s = 1，则 Global Color Table 包含 4 个颜色。
- `Background Color Index`：表示 GIF 的背景色在 Global Color Table 中的索引。
- `Pixel Aspect Ratio`：表示用于计算原始图像中像素宽高比的近似因子，一般情况为 0，顾不做深入了解。

对于我们是示例图，其 Logical Screen Descriptor 对应的字节如下：

    0A 00 0A 00 91 00 00
    
其中：
`Canvas Width` = 0A00 = 10（十进制）
`Canvas Height` = 0A00 = 10（十进制）
`Packed Field` = 10010001（二进制），其中 Global Color Table 为 1，则存在 Global Color Table。Color Resolution 为 1，表示三原色分别用 2 位来表示。Sort Flag = 0，不排序。Global Color Table 中颜色数为 4 。
`Background Color Index` = 0，说明此 GIF 的背景色为 Global Color Table 中第一个颜色。
`Pixel Aspect Ratio` = 0,可忽略。

# Global Color Table

Global Color Table 如果有的话就会跟在 Logical Screen Descriptor 块后面。其块中的字节格式如下：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-49686736a5ecb836.webp)

在 Global Color Table 中每个字节仅代表一种颜色，所以 Global Color Table 的字节数 = 颜色数 * 3.在 Logical Screen Descriptor 中我们知道示例中包含 4 中颜色，即 Global Color Table 的字节数为 12 。所以读取接下的 12 个字节。其具体字节如下：

    FF FF FF FF 00 00 00 00 FF 00 00 00 

根据上面的数据我们来构建 Global Color Table

|索引|字节组合|颜色|
|---|---|---|
|0|FFFFFF|白色|
|1|FF0000|红色|
|2|0000FF|蓝色|
|3|000000|黑色|

# Graphics Control Extension

这里先不说 Graphics Control Extension，我们先看 Global Color Table 后面紧跟的那个字节，从示例中可以看到的`21`，21在 GIF 格式中是有特殊意义的，它表示 Extension Introducer（扩展入口），即接下来的一段数据为最开始提到的这几个扩展中的某一个扩展。

OK，那我们接着 21 往后看，下一个字节为`F9`，F9 也是有特殊含义的，表示这是一个 Graphics Control Extension。

Graphics Control Extension 算上 21 和 F9 一共有八个字节，其结构如下图：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-ebc764ba29f365c1.webp)

其中前两个字节上面已经提到，看接下来的几个字节分别表示什么含义：

- `Byte Size`：表示接下来的有效数据字节数。
- `Packed Field`：是一个包装字段，内部不同位的意义也不同。
    - 从左边数一，二，三位表示Reserved for Future Use，即保留位，暂无用处。
    - 从左边数四，五，六位表示 Display Method，表示在进行逐帧渲染时，前一帧留下的图像作何处理：0：不做任何处理。1：保留前一帧图像，在此基础上进行渲染。2：渲染前将图像置为背景色。3：将被下一帧覆盖的图像重置。
    - 从右数第二位表示 User Input Flag，表示是否需要在得到用户的输入时才进行下一帧的输入（具体用户输入指什么视应用而定）。0 表示无需用户输入。1 表示需要用户输入。
    - 最右边一位，表示 Transparent Flag，当该值为 1 时，后面的 Transparent Color Index 指定的颜色将被当做透明色处理。为 0 则不做处理。
- `Delay Time`：表示 GIF 动图每一帧之间的间隔，单位为百分之一秒。当为 0 时间隔由解码器管理。
- `Transparent Color Index`：当 Transparent Flag 为 1 时，此字节有效，表示此索引在 Global Color Table 中对应的颜色将被当做透明色做处理。
- `Block Terminator`：表示 Extension 到此结束。

下面看一下示例中 Graphics Control Extension 对应的字节：

    21 F9 04 00 00 00 00 00

其中 21，F9 表示这是一个 Graphics Control Extension 块。
Byte Size 为 4。
其它值都为 0 ，概括来讲此 Graphics Control Extension 对应下一帧的渲染无需任何处理，也不需要用户输入，也没有需要做透明处理的颜色值。渲染器要做的就是直接把下一帧图像渲染在画布上即可。

# Image Descriptor

上面讲到 21 上 Extension 的标识符。这里的 Image Descriptor 也有自己的标识符，为 `2C`。下面看一下 Image Descriptor 内部字节结构：

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-607ad12f91e24c47.webp)


其中 Image Seperator 为固定值 2C。

- `Image Left`：该值表示下一帧图像渲染位置离画布左边的距离（从 0 开始）。
- `Image Top`：该值表示下一帧图像渲染位置离画布上边的距离（从 0 开始）。
- `Image Width`：该值表示下一帧图像的宽度。
- `Image Height`：该值表示下一帧图像的高度。
- `Packed Field`：这是一个包装字段，内部不同位的意义也不同。
    - 从左数第一位：Local Color Table Flag，表示下一帧图像是否需要一个独立的颜色表。1 为需要，0 为不需要。
    - 从左数第二位：Interlace Flag，表示是否需要隔行扫描。1 为需要，0 为不需要。
    - 从左数第三位：Sort Flag，如果需要 Local Color Table 的话，这个字段表示其排列顺序，同 Global Color Table。
    - 从左数第四、五位：Reserved For Future Use，保留位。
    - 从左数最后三位：Size of Local Color Table，同 Global Color Table 中的该位。如需要本地颜色表，则该数有效。
    
接着看一下示例中的对应字节：

    2C 00 00 00 00 0A 00 0A 00 00 

2C 表示 Image Descriptor
Image Left = 0
Image Top = 0
Image Width = 10
Image Height = 10
上面这四个数字表示即将渲染的一帧大小为 10*10 像素，正好与 GIF 图的分辨率一致。
打包字段都为零，表示下一帧不需要 Local Color Table，也不需要进行隔行扫描。

> 这里读者可能会好奇我们不是在 Logical Screen Descriptor 中知道了图像的分辨率吗，为什么还要在 Image Descriptor 中额外指定图像的宽和高。其实 GIF 在进行编码的时候并不一定对每一帧进行全尺寸的压缩。因为有时候一个 GIF 图只有中间区域是动的，四周都是静止的，那只需要对中间那部分进行压缩编码即可。所以这里的 Image Left、Image Top、Image Width 和 Image Height正好可以指定一个小于等于 GIF 分辨率的图像。


# Local Color Table
local color table 在本示例中不涉及，我也不多介绍，在处理的时候按照 Global Color Table 处理即可。

# Image Data

如果存在 Local Color Table，Image Data 就紧跟其后。如若不存在，则紧跟在 Image Descriptor 后。下面先看一下 Image Data 的内部字节组成。

![image](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/2702499-8b4ded747247ce4a.webp)

- `LZW Minimum Code Size`: GIF 在对每一帧的像素颜色在 Color Table 所对应的索引进行 LZW 压缩，这里的 LZW Minimum Code Size 就是 LZW 压缩中很关键的一个值，不过目前这个值先放着，等后面讲到对 LZW 解压缩时再讲。
- `Number of bytes of data in sun-blocks（01-FF）`：这个值表示在其后面的有效字节的个数。范围为 01-FF，当其值为 0，则表示 Image Data 到此为止，后面就是其他块的数据了。这里需要注意由于其最大值为 FF，但图像的像素个数可能会大于这个值，所以从图上也能知道这个 Data sub-Blocks是有可能接连出现很多个的。
- `Data Sub-Block(s)`：表示有效的字节块。
- `Block Terminator`：表示 Image Data 的结束部分。

接着看一下示例中对应的字节：

    02 16 8C 2D 99 87 2A 1C DC 33 A0 02 75 EC 95 FA A8 DE 60 8C 04 91 4C 01 00
    
02 表示 LZW Minimum Code Size。
16（十六进制） 表示后面紧跟着的 22 个字节用来表示下一帧的图像数据。
00 表示 Image Data 到此为止。

# Trailer

从上面的示例看，最后还剩下一个字节 3B，这个在 GIF 中也有特殊含义，是尾部标记的意思，GIF 的字节内容到此就结束了。

# Plain Text Extension、Application Extension和Comment Extension

最后还剩下上面三个 Extension，他们主要是为 GIF 提供一些额外的信息，本身的信息对实际的渲染没有多少影响。所以这里我也不多介绍，想深入了解的可以阅读一开始提到的两个网址。

# 总结
本文主要以 GIF 内部字节的格式作为出发点，简单介绍了十一种块。只有充分理解了各个块内部的含义，才能为其编写正确的解码器。但仅仅只了解各个块还是不够的，GIF 的图像数据采用的是 LZW 算法进行压缩，所以还需要对 LZW 有较深理解。在下一篇《GIF 与 LWZ》中我将结合本文中的示例图像，详细讲解如何通过 LWZ 对 GIF 的图像数据进行压缩和解压。