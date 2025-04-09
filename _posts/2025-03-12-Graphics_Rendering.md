---
layout: post
title: Computer Graphics (一) 手动实现软渲染
subtitle: 软渲染是图形学基础中的基础
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
excerpt_image: /assets/images/GraphicsRendering/ShadowOffset43.png
tags: 图形渲染
priority: 1
top: 1
---

![banner](/assets/images/GraphicsRendering/ShadowOffset43.png)  

在现代渲染流水线中, [图形渲染](https://blog.csdn.net/jinking01/article/details/105389977)的大部分流程直接由硬件(GPU)完成, CPU 负责管线中可编程的部分. 例如最常见的 Shader 和一些固定的参数.  

软渲染指的是由 CPU 模拟图形渲染的流水线, 这样能够加深对图形渲染流水线的理解. 本项目来源 Github 开源项目 [TinyRenderer](https://github.com/ssloy/tinyrenderer), 作为实践过程中的学习笔记以及一些个人的理解.  

---  

## 简介
TinyRenderer 是一个极其简化的软渲染程序, 更加直观的说法, 就是负责将空间中的三角形转换为屏幕上的像素. 所以核心基本上就是三维空间转换加上光栅化.  

在现代图形 API, 例如 OpenGL 或者 DirectX, 这两个流程都是内部直接完成的. TinyRenderer 展示了这两个流程的基本原理, 帮助那些对于现代图形 API 工作机制有疑惑的新手.  

---  

## TGAImage
TGAImage 是读写 .[tga 格式文件](https://www.cnblogs.com/fortunely/p/18022053)的功能库. TGA 格式是  

该功能库的作用就是从 tga 格式文件中, 解析它的格式数据, 按照对应的格式读写 Image Data.  

> 主要疑问点集中在 [Image Origin](https://stackoverflow.com/questions/25113149/in-the-tga-file-format-what-is-the-purpose-of-the-x-origin-and-y-origin). 这个字段并不是我们所想的图像的原点
> 真正的图像原点设置在 ImageDescriptor 中的 Bit5 和 Bit4   
> windows 操作系统的屏幕空间以左上角为原点. 数学坐标系中以左下角为原点. 并且大部分图像文件格式都是左下角为原点.  
> 左上角为原点, 符合人类阅读顺序的图像文件内存布局, 从上往下, 从左往右读, 因此左上角为原点.
> 左下角为原点, 符合数学计算.  

---  

## 渲染三角形
图形渲染的第一步, 就是渲染一个三角形到屏幕上. 手写软渲染时, 就发现在屏幕上显示一个三角形看上去简单, 实际上涉及了很多图形学基础知识. 需要对渲染流水线有着基础但完整的了解.  

最开始学习如何绘制两个顶点之间的屏幕线条, 以像素为单位的屏幕坐标可以视为离散坐标, 无法真正实现数学意义上的线条. Bresenham 算法算是简单易用的线条绘制方法.  

---  