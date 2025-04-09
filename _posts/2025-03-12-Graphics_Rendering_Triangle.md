---
layout: post
title: Computer Graphics (二) 手动实现软渲染-三角形
subtitle: 绘制点 线 三角形
author: LM
categories: 图形渲染
# banner:
#   # video: https://vjs.zencdn.net/v/oceans.mp4
#   #loop: true
#   #volume: 0.8
#   #start_at: 8.5
#   image: /assets/images/banners/FlipModel.png
#   opacity: 0.8
#   background: "#000"
#   height: "50vh"
#   min_height: "38vh"
#   heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
#   subheading_style: "color: gold"
excerpt_image: /assets/images/GraphicsRendering/scanline1.png
tags: 图形渲染
priority: 2
---

![banner](/assets/images/GraphicsRendering/scanline1.png)  

每个图形 API 的第一课都是渲染一个三角形到屏幕上, TinyRenderer 也不例外. 相比于在游戏引擎中渲染一个三角形, 使用 OpenGL 或者 DirectX 渲染三角形要复杂的多, 手写的软渲染只会更加复杂.  

---  

## 点
TinyRenderer 项目中绘制点是后续所有功能的基础. 绘制点的易错点主要在于 tga 格式文件与屏幕坐标系的差异.  

tga 格式文件默认 Image Origin 是左下角. <span style='color:#4cd137'>图像文件主要涉及的是内存中数据的布局与屏幕坐标系布局的转换</span>  

个人理解:  
1. 从内存中读取数据的顺序是固定从小地址到大地址的, 想象我们的阅读一本书的顺序, 很自然的从上往下, 从左往右. 
2. tga 格式规定了默认的 Image Origin 位于左下角. 等于规定了内存数据小地址到大地址映射到图像中是从下往上, 从左往右  
3. 相当于 tga 文件自己定义了一个屏幕坐标空间, 规定原点以及xy轴正向方向.
4. 图像处理程序读取头文件就可以了解 tga 文件定义的屏幕坐标空间, 然后将屏幕坐标转换到当前环境使用的屏幕坐标空间

## 线
图形渲染流水线的最初输入数据就是顶点数据, 最简单的情况下, 每三个顶点确定一个三角形. 绘制一个三角形的方法就是, 绘制三个顶点之间的线.   

### First attempt
```cpp 
void line(in x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    for(float t = 0; t < 1; t+=.01)
    {
        int x = x0 + (x1 - x0) * t;
        int y = y0 + (y1 - y0) * t;
        image.set(x, y, color);
    }    
}
```
t作为比例系数，取值代表起点和终点之间的间隔点，必须足够小，不然点的分布稀疏不足以连成线

### Second attempt
首先，每一个像素占用二维空间中的一个坐标，因此，小数点坐标是没有意义的，比如(0.1,2.1)。我们使用x坐标，每次递进1，计算出对应的y坐标。
```cpp 
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    for(int x = x0; x <= x1; x++)
    {
        float t = (x - x0) / float(x1 - x0);
        int y = y0*(1.0 - t) + y1 * t;
        image.set(x, y ,color);
    }
}
```  


在斜率小于1时，运行正常，<span style='color:#4cd137'>当斜率大于1时，也就是x步进1，y步进大于1，这时就会出现裂缝。</span>  

---  
 
## BresenhamLine
斜率小于1，本质上是保证在x轴上步进一个像素时，y轴上步进小于2，这样 像素才能连接起来。因此，我们要保证自变量步进1时，因变量步进小于2.我们肯定不能更改要绘制线的斜率，但是我们能更改自变量和因变量。
```csharp 
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color)
{
    // 陡峭，意味着dy/dx > 1 => dx < dy
    bool steep = false;
    if(std::abs(x0-x1) < std::abs(y0-y1))
    {
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if(x0 > x1)
    {
        // 保证从左到右绘制，不会影响绘制的坐标
        std::swap(x0, x1);
        std::swap(y0, y1);
    }
    for(int x = x0; x <= x1; x++)
    {
        // t是比例
        float t = (x-x0)/(float)(x1 - x0);
        int y = y0(1.-t) + y1*t;
        if(steep)
        {
            // 再反转一次
            image.set(y,x,color);
        }else{
            image.set(x,y,color);
        }
    }
}
```  

三角形三个顶点组成三个边, 使用 BresenhamLine 就可以绘制出一个三角形, 但是好像和通常图形学入门第一课的三角形不同.  
这种只有边框的三角形通常称为线框模式, 而使用 OpenGL 或者 DX 渲染的三角形通常都是填充三角形.  