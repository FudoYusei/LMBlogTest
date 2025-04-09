---
layout: post
title: Flip Model(九) WaitEvent
subtitle: Flip Model 中的等待事件
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
priority: 9
---

![banner](/assets/images/banners/FlipModel.png)  

wait for event 的本质就是同步机制, 如上图所示, 在 FlipModel Sample 渲染流程中对应的同步点:  
+ vSync: 基于显示器 vBlank 进行的同步机制, 等待显示器周期性的 vBlank 信号
+ Wait For Present: 等待 `SwapChain::Present` 方法返回
+ Wait For BackBuffer: GPU 执行渲染命令, 需要等待可用的 BackBuffer
+ Wait For GPU: 等待 GPU 渲染完毕
+ Wait For WaitableObject: 等待 PresentQueue 有空余位置

---  

## vSync
在 [FlipMode Sample](IntelSample) 中并没有代码明确的等待 vBlank 进行垂直同步.  
<span style='color:#4cd137'>垂直同步是由 SwapChain 关联的 DXGI 内部机制自动实现的, 可以通过 SwapChain 的某些参数改变内部机制</span>  

> 好像并没有明确提供关于 vSync 的 API, 但是可以从 `DXGI_FRAME_STATISTICS` 这个数据统计中间接获取 vSync, 后续会详细解释围绕着 vSync 的机制   
 
---  

## Wait For BackBuffer
这是一个内部机制, 并没有明面上的显示, 由 SwapChain 关联的 PresentQueueManager(个人称呼) 自动管理. 显示器当前正在显示的 Buffer, 可以称为 FrontBuffer, 其余都是 BackBuffer, 应用程序可以使用 BackBuffer 预渲染帧.  
但是当 BackBuffer 都写满了, <span style='color:#4cd137'> PresentQueueManager 就会延迟队列中 PresentFrame 的 GPU 渲染直到显示器进入下一个 vBlank</span>  
  
---  

## Wait For GPU  
CPU 生成渲染指令时, 会使用 `Fence` 机制同步 CPU 和 GPU. 类似于 GPU 需要 BackBuffer 才能领先于显示器的进度进行预渲染.  
CPU 需要 FrameBuffer 才能领先于 GPU 预生成渲染指令, Frame 中会包含每一帧渲染所需要的资源.  

---  

## WaitForPresent
WaitForPresent 并不是事件同步机制, 在应用程序调用 `SwapChain::Present` 之前记录开始时间, 方法返回之后记录结束时间.  
也就是说, WaitForPresent 记录的是这个方法的总耗时, 这里的 Present 是指的就是应用程序当前渲染逻辑帧入列操作  

> 这里的 Present 操作是 Present To PresentQueue, 与 Present To Monitor 不同. 当 PresentQueue 满载时, `SwapChain::Present` 会阻塞, 直到队首出列  

---  

## Wait For WaitableObject  
`SwapChain::Present` 表示 CPU 端的渲染工作完成, 一旦发生阻塞, 那么就会增加帧的显示延迟, 需要尽量减少帧生成后到 Present To Monitor 之间的时间.   
阻塞是由于 Present To Queue 时, 队列已满引起的. 那么只要在队列满载时, 推迟帧生成. 相当于把等待队列的时间移动到外部, 等待时间确实没有消失, 只是转移.  

---  

