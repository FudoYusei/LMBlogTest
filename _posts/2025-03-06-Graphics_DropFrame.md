---
layout: post
title: Flip Model(八) Dropped Present
subtitle: 丢弃的帧
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
priority: 8
---

![banner](/assets/images/banners/FlipModel.png)  

Dropped Present 字面意思是丢弃的 Present 操作, 在 FlipModel 中, 被丢弃的帧时机含义是 PresentQueue 出列后, 未在显示器上显示的帧. 看起来像丢弃了一样, 但是 `DXGI_FRAME_STATISTICS::PresentCount` 依然记录了帧的出列记录. <span style='color:#4cd137'>如同大部分数据删除一样, 只是隐藏数据, 并不会直接删除, 依然保留数据存在的记录.</span>  
  

初次看到 DropFrame 会感觉很奇怪, 垂直同步情况下, 应用程序会和显示器刷新率同步, 只有 vBlank 才会进行一次 `SwapChain::Present`, 本质上是只有 vBlank, PresentQueue 才会进行一次 PresentToMonitor 操作. <span style='color:#fbc531'>那么只有在垂直同步关闭时, 才可以随意调用 `SwapChain::Present`</span>  
  
但是结合之前关于 FlipModel 垂直同步的分析, 可以得知 FlipModel 就算设置 `SwapChain::Present()` 的 SyncInterval 为 0, 并不是真正的关闭垂直同步, 依然只会在 vBlank 时进行一次或者多次 PresentToMonitor 操作. 但是 PresentQueue 中的 SyncInterval 为 0 的帧(后续称为零帧) 拥有抢占和被抢占的特性  

> 除非同时设置 DXGI_PRESENT_ALLOW_TEARING, 才是传统意义上的 VSync OFF  

## DropFrame 行为  
FlipModel SyncInterval 设置为0, 在 vSync Interval (两次 vBlank 期间) 多次调用 SwapChain::Present API 提交多帧, SwapChain 会选用最近提交的那一帧 Present to monitor, 剩余的帧都会被丢弃?  

应用程序中的 PresentQueue 是对应的是 CPU 端调用 SwapChain::Present API 时提交的逻辑帧 ,并没有明确的文档相关. 但是可以通过设置 SwapChain 相关的参数来配置 DXGI 对于 PresentQueue 的管理行为.    

默认情况下, 可以观察到的行为:   
1. 显示器只会以固定周期刷新画面, 并不在意显示内容究竟指向哪个 Buffer, 哪怕刷新过程中, 更换指向 Buffer 都可以.
2. <span style='color:#4cd137'>无论是否开启垂直同步, 都会等待至 vBlank 时</span> , 才会开始执行 PresentToMonitor 操作. (除非设置 DXGI_PRESENT_ALLOW_TEARING)
3. 当执行 PresentToMonitor 操作时, 会从 PresentQueue 队首开始执行, 队首零帧具有特殊的抢占和被抢占特性
4. 因此, 在同一次 vBlank 周期里, PresentQueue 中缓存了多个零帧, PresentQueue 会一直执行 PresentToMonitor, 直到队首不是零帧(或者队列为空)

## DropPresent 机制   
已知 DropFrame 的机制:  
1. 应用程序在一次 vBlank 到来前提交多个零帧(VSyncInterval = 0), PresentQueue 中会充斥多个零帧
2. 在 vBlank 期间, 由于零帧的抢占和被抢占特性, PresentQueue 会多次执行 PresentToMonitor  
3. `DXGI_FRAME_STATISTICS::PresentCount` 会跟随 PresentToMonitor 次数增加, 相当于主键 ID
4. 最终可以在显示器上显示的帧 PresentID 就是 PresentCount, 其他的都是 DropFrame  

那么检测 DropFrame 方法:  
1. `LastRetrivedID` 记录上次 VSync 时, 最终显示在显示器上的帧的 PresentID  
2. 这次 VSync 时, 记录最终显示在显示器上的帧的 PresentID  
3. 两者之间的 PresentID 代表的帧自然就是 DropFrame

## DropPresent 和 Glitch 的区别
Glitch 是渲染延迟, 通过 PresentID 匹配渲染逻辑帧, 然后对比当前真正同步的次数与预计的同步次数之间的差值. 超过缓冲区机制延迟则说明发生 Glitch, 为了追上延迟的帧数, 需要立即释放 Present Queue 中对应 Glitch Count   

Dropped Present 记录两次同步之间被跨越的操作.  

## 应用程序与 VSync  
VSync 机制的本质是:  
1. PresentQueue 只会在 VBlank 期间进行 PresentToMonitor 同步操作
2. 导致 PresentQueue 也只在 VBlank 期间才会出列. 
3. 应用程序调用 `SwapChain::Present` 才不会堵塞.  

最终的效果就是, VSync 机制会让应用程序的帧率与显示器刷新率一致. 那么关闭垂直同步, 为什么会让应用程序帧率上升呢?  

<span style='color:#4cd137'>垂直同步关闭(全零帧)</span>  
1. PresentQueue 依然只会在 VBlank 期间进行 PresentToMonitor 同步操作
   1. 因为零帧的特性, PresentQueue 会循环出列, 直到队列为空, 或者出列帧不是零帧.
   2. 全零帧的情况下, 每次 PresentToMonitor 都会清空 Present Queue
2. 在下一次 VSync 到来前, 只要 PresentQueue 未满列, 应用程序可以多次调用 `SwapChain::Present`  

可以看到应用程序最大帧率是 PresentQueue.Count * RefreshRate  

<span style='color:#4cd137'>垂直同步关闭(DXGI_PRESENT_ALLOW_TEARING 加上全零帧)</span>  
1. PresentQueue 队首帧一旦 GPU 渲染完毕, 立即进行 PresentToMonitor 同步操作
2. 理想状态下, 只要渲染速率够快, PresentQueue 就不会满列, 应用程序帧率无限, 实际情况, 帧率受到 CPU 和 GPU 渲染速率限制, PresentQueue 中会堆积 GPU 未来得及渲染的 Present 帧.  

