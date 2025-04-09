---
layout: post
title: Flip Model(一) 概览
subtitle: Direct3D Presentation Flip Model
author: LM
categories: Direct3D
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
excerpt_image: /assets/images/banners/FlipModel.png
tags: Direct3D 图形学 渲染
top: 1
priority: 1
---

![banner](/assets/images/banners/FlipModel.png)

有关图形渲染管线的优秀教程很多, 不过大多是阐述整体概念性的流程, 跟着这些教程能得到对渲染流程整体的认知. 但是对于细节上的实现基本完全没有解释, 导致跟完了教程之后, 真正想要做出实用的东西, 就会两眼一抹黑.   

> 本质上, 细节太多太复杂, 可能想要解释清楚一个细节都要比教程本身都长. 所以有种说法, 想要衡量一个开发的技术水平, 只要问问: <span style='color:#4cd137'>踩了多少坑, 踩得有多深</span>  

这个博客的起点在于, DirectX (OpenGL) 的渲染教程中, 对于 `ISwapChain::Present` (`SwapBuffers`) 这个 API 的疑问: <span style='color:#fbc531'>这个 API 告知显示系统 BackBuffer 已经准备好, 可以 Present To Monitor.</span>     
1. 当前帧的 BackBuffer 准备好, 那么 GPU 必然渲染完毕当前帧, 但是又说这条指令并不会让 CPU 与 GPU 同步, 这条指令并不会阻塞 CPU, 不会等待到 GPU 渲染完成才返回.
2. Direct3D12 使用 Fence 机制同步 CPU/GPU, OpenGL 使用 `Flush()` 同步 CPU/GPU. 并且通常是在 `Present` 方法之后调用, 更说明 `Present` 方法不负责 CPU/GPU 同步.
3. 在使用 `Present` API 之后, CPU 可以直接使用 `SwapChain::GetBuffer()` 得到 BackBuffer, 并且继续渲染下一帧. 也就是说 `Present` API 就已经翻转了 Buffer, 但是 GPU 还可能未渲染, 帧也没有 Present To Monitor.  
 

<span style='color:#4cd137'>SwapChain 作为 CPU GPU 以及显示子系统的连接点, 内部机制远比看起来复杂的多. CPU GPU 以及 显示子系统之间的同步机制也很复杂.</span>  
   
在网上几乎没有与 SwapChain 相关机制的介绍, 甚至连提问的都很少, 求助于 AI, 得到的也是一些似是而非的回答. 不过最终还是找到的是 Intel 在早年发表的一篇技术文章 Present Latency, DWM and Waitable Swapchains and FlipModel, 以及附带的 FlipModel 工程文件.   
这个项目使用了 Direct3D 12 新增的 FlipModel 渲染流水线, 并且做了可视化处理. 通过阅读代码, 能够反推部分 SwapChain 的机制, 并且可以了解到游戏中常见的参数计算方式, 例如: 
+ 帧延迟 
+ CPU 延迟 
+ GPU 延迟 
+ 垂直同步 
+ HighRespective
+ Present 延迟

## Flip Model
Flip Model 是微软随着 win10 一起推出的新的 SwapChain 翻转模型, 虽然微软官方极力推崇 [For Best Performance Use dxgi flip model](use-flipmodel). 但事实上, 支持 D3D12 的游戏都很少, D3D12 虽然给了开发人员更为底层的控制力度, 相应的, 需要学习的东西更多, 如果是熟练使用 D3D11 的, 能够很快上手. 就我个人学习经历来看, 纯新手大多数淹没在 D3D12 天书一样的 API 中.  

> D3D12 更像 D3D11 的进阶版, 通过比对两者的差异, 确实能够搞明白一些更为底层的机制. 就是完全不适合新手, 至少要能熟练使用 D3D11.  

<span style='color:#4cd137'> Flip Model 最大的优势是: 将无边框全屏窗口模式优化成了与全屏独占一样的性能</span>  

想要使用 Flip Model, 看似简单的修改一个参数即可, 但是内部机制差别就大了.  
```cpp
swap_chain_desc.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
```

[use-flipmodel]: https://learn.microsoft.com/zh-cn/windows/win32/direct3ddxgi/for-best-performance--use-dxgi-flip-model  

## 性能参数
游戏中经常能看到设置选项: 
+ 垂直同步 
+ 双缓冲 
+ 三缓冲  

性能检测软件中经常能看到的显示条目:  
+ FPS 程序帧数
+ CPU FPS CPU 帧数
+ GPU FPS GPU 帧数
+ low FPS 最低怎书
+ CPU 延迟 CPU 延迟
+ GPU 延迟 GPU 延迟 
+ 帧延迟 
+ 显示延迟 帧生成到显示的延迟 

网上的解释基本也是概念性的解释, 因为这些概念实际上与渲染管线深度绑定, 它们本质上就是一套完整的渲染流程的各个部分的展示. 单独解释其中一部分, 是无法窥一斑而知全身, 反而容易得出错误的结论.  

## 总结
这一系列的博客会基于 DirectX 实现的一个简单的图形程序, 来阐述游戏中常见的各种性能参数的实际意义, 接下来会逐一对各种性能参数进行单独和深入的解释.  