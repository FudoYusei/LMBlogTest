---
layout: post
title: UnityRendering (一) 概览
subtitle: Unity3D 渲染相关知识
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_Overview.png
tags: UnityRendering 
priority: 1
top: 1
---

![banner](/assets/images/UnityRendering/UnityRendering_Overview.png)  

Unity3D 渲染流程中采用的是标准的 OpenGL 风格的矩阵计算. 介绍这些标准管线的文章很多, 本文尝试在 Unity3D 中使用 Shader 来实践渲染管线的理论知识.  

---  

## 左右手定则
左右手定则是人为规定的理论, 例如右手定则可以用于描述电磁场与电流方向, 左右手定则相当于为一系列的数学规律定义了手性. 没有手性, 这些数学规律依然存在.  
左右手定则实际上是将数学规律描述成人能观察到的现象, 方便使用和交流. 因此将左右手定则当作定理记住就行了.  

<span style='color:#fbc531'> Unity3D 明显使用的是左手坐标系, 而 OpenGL 使用的是右手坐标系.</span>   
但是可以明确的说, 在 Unity3D 内部涉及到渲染的部分都是使用的 OpenGL 标准. 这里所说的左右手坐标系实质上指的是世界坐标系.  

<span style='color:#4cd137'>同样, DirectX 左手坐标系, OpenGL 右手坐标系都是指的世界坐标系. 在搭建渲染场景时, 以观察者视角规定出来的左右手坐标系(人为规定的 Right Front Up).</span>    

当没有基准坐标系时, 所谓的左右手坐标系或者偏手性都是没有意义的  

举例来说, 三个向量 (0, 0, 1) (0, 1, 0) (1, 0, 0), 这三个向量在左手坐标系中组成的空间就是左手坐标系, 在右手坐标系中组成的空间就是右手坐标系.   
但是, 现在以左手坐标系为基准, 三个向量 (0, 0, 1) (0, 1, 0) (1, 0, 0) 组成的就是左手坐标系, (0, 0, 1) (0, 1, 0) (-1, 0, 0) 组成的就是右手坐标系.  

---  

## OpenGL-like API  
Unity3D 渲染流程采用的是 OpenGL 风格. 具体表现在哪?  

最本质的区别就是 NDC 和屏幕坐标系.  
+ 虽然 NDC 的范围也可以进行设置, 但很少会这么做. OpenGL 的范围是 [-1, 1] DX 的范围是 [0, 1]  
+ 屏幕坐标系无法设置, NDC 通过 viewport 矩阵转换为屏幕坐标, OpenGL 屏幕坐标系原点在左下角, DX 屏幕坐标系在左上角, 这也是经常在渲染代码中看到 y 轴翻转  

---  

## 旋转正方向  
旋转正方向也是常见的理论, 左手坐标系的旋转正方向是顺时针, 右手坐标系的旋转正方向是逆时针. 描述中隐藏了一个条件, 就是观察者在z轴正方向的位置朝着原点看过去旋转的方向.  

旋转方向应该是和叉乘计算有关, 用左右手定则表现出来就是顺逆时针.  

---  

## 正反面
已知三角形三个顶点, 如何判断三角形正反面?  
在左手坐标系中, 顶点顺时针排列为正面, 顶点逆时针排列为反面  
在右手坐标系中, 顶点逆时针排列为正面, 顶点顺时针排列为反面  
正反面实际上与旋转正方向本质上相同, ABC 三个点组成面的法向量 $ \vec{n} = \vec{AB} \times \vec{AC} $ 法向量指向观察者, 说明是正面否则是反面, 同样也是依赖于叉乘的规则, 因此需要遵守左右手定则.  

---  


