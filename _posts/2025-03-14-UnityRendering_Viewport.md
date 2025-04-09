---
layout: post
title: UnityRendering (九) Viewport
subtitle: 容易被忽略的 Viewport
author: LM
categories: UnityRendering
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_Viewport.jpeg
tags: UnityRendering
priority: 9
---

![banner](/assets/images/UnityRendering/UnityRendering_Viewport.jpeg)  

标准渲染流程的讲解很容易让人忽略 Viewport. 直到遇到了 Unity3D 中 FlipY 的问题仔细研究了 Viewport 设置.  

---

## Viewport
Viewport 可以简单的理解为表示显示区域的矩形范围, 每一次渲染流程必须要设置. 渲染流程会使用 Viewport 矩阵将 NDC 坐标转换成屏幕坐标系的坐标.  

```c
// 设置一个矩形区域
glViewport(x, y, width, height);
```  
通俗的来说就是在显示器上找到一块矩形区域, 将渲染结果显示在这片显示区域中.  

DX 默认屏幕坐标的原点处于左上角, 而 OpenGL 默认屏幕坐标系的原点处于左下角, 因此设置的 ViewportX 与 ViewportY 含义不同.  
```c
// directx 定义 viewport 矩形 左上角屏幕坐标与对应宽高
DXViewport(LeftTopX, LeftTopY, Width, Height);

// opengl 定义 viewport 矩形 左下角屏幕坐标与对应宽高
GLViewport(LeftBottomX, LeftBottomY, Width, Height);

```  

### NDC
NDC 坐标系基本是一致的, z 值的范围可能不一样. 但是 xy 的范围都是 [-1, 1]  
+ (-1,1) 左上角
+ (-1,-1) 左下角
+ (1, 1) 右上角
+ (1, -1) 右下角  

将四个角落顶点转换为屏幕坐标系, 可以清晰看到转换过程.  

### DirectX 
https://learn.microsoft.com/zh-cn/windows/win32/direct3d9/viewports-and-clipping  

从微软文档中, 可以得出 Viewport 依然是一个映射矩阵, 这个矩阵看上去简单, 实际上隐含了向屏幕坐标系的转换.  

<span style='color:#4cd137'>注意 DX 是向量左乘, </span> 对应的四个角落坐标转换:  
+ (-1, 1) NDC 左上角 => (LeftTopX, LeftTopY) viewport 左上角
+ (-1, -1) NDC 左下角 => (LeftTopX, LeftTopY + Height) viewport 左下角
+ (1, 1) NDC 右上角 => (LeftTopX + Width, 0) viewport 右上角
+ (1, -1) NDC 右下角 => (LeftTopX + Width, LeftTopY + Height) viewport 右下角


### OpenGL
OpenGL 风格的 Viewport 矩阵, viewport 原点在左下角  

<span style='color:#4cd137'>OpenGL 是向量右乘, </span> 对应的四个角落坐标转换:  
+ (-1, 1) NDC 左上角 => (LeftBottomX, LeftBottomY + Height) viewport  左上角
+ (-1, -1) NDC 左下角 => (LeftBottomX, LeftBottomY) viewport 左下角
+ (1, 1) NDC 右上角 => (LeftBottomX + Width, LeftBottomY + Height) viewport 右上角
+ (1, -1) NDC 右下角 => (LeftBottomX + Width, LeftBottomY) viewport 右下角  

### 总结
无论 DirectX 还是 OpenGL 都是将 NDC 坐标转换为自己预设的屏幕坐标系中.  
<span style='color:#fbc531'>问题就是这个预设的屏幕坐标系与实际使用的坐标系可能不同</span>  
例如, windows 平台的图形程序统一屏幕坐标系为左上角. 所以 DX 默认屏幕坐标系为左上角, 大部分图片格式也是同样默认原点为左上角.  

<span style='color:#4cd137'>我们使用 DX 或者 OpenGL 只需要考虑如何将内容正确的渲染到预设的屏幕坐标系中, 至于屏幕坐标系的内容如何正确的显示在不同的系统中, 由 DX 和 OpenGL 的底层驱动来保证.</span>  
