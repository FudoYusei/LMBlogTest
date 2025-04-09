---
layout: post
title: Flip Model(二) 性能参数
subtitle: Present Latency, DWM and Waitable Swapchains
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
priority: 2
---

![banner](/assets/images/banners/FlipModel.png)

## Monitor Works
显示器是显示子系统的输出终端, 这里不严谨的使用显示器代表显示子系统中的软件和硬件.  

在渲染流程中, 显示子系统或者说显示器在渲染流程中的功能是固定的, 按照自身的刷新率周期性的发出 vBlank 信号, vBlank 结束后, 读取显示缓冲区中的内容, 刷新显示器的画面.  

> vSync 是基于 vBlank 实现的同步机制, 同步应用程序与显示器的帧率, 并不是显示器的设置. 
> 无论垂直同步开启还是关闭, 显示器都只是周期性的重复这个过程 (不考虑 g-sync 技术).

---  

## 参数解释
[Intel Sample][IntelSample] 是基于 D3D12 SwapChain 实现一个简单的图形程序, 将主要参数对渲染流程的影响进行直观的图形化显示.  

### FullScreen  
window 和 FullScreen 在 Windows 操作系统中的渲染流程不同 , Windows 操作系统中中有一个桌面管理系统 DWM. 默认情况下, DWM 统一管理系统中所有的窗口显示.  

<span style='color:#4cd137'>窗口化应用程序提交渲染内容的输出对象并不是显示器, 而是 DWM 的 BackBuffer. DWM会组合所有将要显示在桌面上的窗口的渲染内容, 然后将这些内容提交给显示器. </span>  

<span style='color:#4cd137'>全屏应用程序不需要 DWM 管理, 后台缓冲区直接提交至显示器</span>  

> 现在的 windows 操作系统还有一个无边框窗口模式, 结合了窗口化与全屏的优点, 有着全屏的性能和窗口化随时能切屏的优点  
> 个人使用感受, 无边框窗口模式性能还是要低一点, 可能是为了随时能切屏, 后台程序并未停止提交渲染帧?

DWM 除了性能问题外, 还天然具有延迟:  
1. 应用准备好渲染的backbuffer, 然后调用present()通知显示系统进行呈现  
2. DWM在VBLANK被唤醒, 获取最新的backbuffer并进行组合. 组合完毕之后, DWM同样在进行类似Present()的操作, 只是这次针对是显示器  
3. 在下一次VBLANK时候, 才真正的显示内容  

所以, 此时已经间隔了两个VBLANK.  

### vSync
扫描枪从显示器底部重置回顶部的时间, 称为 vBlank, 这时是一帧结束还未进行下一帧绘制的空闲时间. 这段时间显示器不会显示新的内容, 利用这段时间, 应用程序与显示器进行帧率同步, 这个机制被称为 vSync.  
两次 vBlank 期间的时间被称为 vSync Interval,   
<span style='color:#4cd137'>虽然现代显示器并不使用扫描枪, 但是这个同步机制还是保留, 并且读取内存内容的方式并未改变, 依然是行扫描垂直刷新的方式</span>  

### MaximumFrameLatency BufferCount FrameCount
这三个参数是渲染内容 Present 的核心参数, 每一个都深度绑定渲染流水线   

+ `MaximumFrameLatency` 最大帧延迟, 这个帧指的是应用程序调用 `SwapChain::Present` 提交的帧, 这里牵扯到 PresentQueue 这个概念, MaximumFrameLatency 可以视为 PresentQueue 的最大容量.  
+ `BufferCount` 缓冲区数量, 在渲染流程中缓冲区被分为 BackBuffer 和 FrontBuffer, FrontBuffer 是显示器要显示的内容, BackBuffer 用于让应用程序提前渲染下一帧.  
+ `FrameCount` 帧缓冲数量, 这个帧指的是应用程序能够缓存的渲染逻辑帧. 用于同步 CPU 和 GPU, 减少帧延迟.  

### Present 和 Waitable Object  
`SwapChain::Present` 机制很复杂, 这里简单的说明: 当应用程序调用这个 API, 会将当前的渲染逻辑帧提交给背后的 DXGI 运行时中的一个 PresentQueue, 成功入列后, 方法返回, 应用程序继续执行.  

<span style='color:#4cd137'>如果应用程序调用 `SwapChain::Present` 时, PresentQueue 满列, 那么应用程序就会阻塞</span>  
Waitable Object 用来防止 `SwapChain::Present` 阻塞, 保证调用前, PresentQueue 不会处于满列状态, 只要在 SwapChain 中开启了 WaitableObject 功能, 并在应用程序每一帧开头, 订阅对应的 PresentQueue 满列事件.   

> CPU GPU PresentQueue 以及显示系统之间都是并行的关系, 因为人为的需求(例如低延迟, 防止画面撕裂等) ,需要在某些操作上同步  

### Present Latency  
Present Latency 比较特殊, 因为 Present 这个操作本身并不是连续性的, 所以无法直接计算出花费的时间. 只能通过结束时间减去开始时间得到这个 Present 操作的延迟.  

1. 每当应用程序调用 `SwapChain::Present` 成功返回时, 意味着当前帧(暂且命名为 PresentFrame)成功加入 PresentQueue, 这个时间点就是当前帧的开始时间.  
2. 当 PresentFrame 被呈现到显示器之后, 记录结束时间.  
3. 两者相减得到的就是 Present Latency(也称为帧延迟)

Present Latency 的原因非常复杂. 需要结合 `SwapChain` 设置的参数与渲染流程中其余的参数才能分析出性能瓶颈.  

### Latency
1. CPU Latency, CPU 从新的一帧渲染指令开始到 `SwapChain::Present` 之前花费的时间
2. GPU Latency, GPU 渲染一帧的时间, 在同一帧渲染指令的开头和末尾打上时间戳, 计算 GPU 渲染的时间, 也就是延迟

一次完整的 GameLoop 所有花费的时间:  
1. 游戏逻辑帧更新花费的时间  
2. 渲染逻辑帧更新花费的 CPU 时间
3. 渲染指令花费的 GPU 时间
4. 渲染完毕的内容提交到显示器上花费的时间
在游戏逻辑更新时, 就要处理用户操作, 从这时候开始直至提交到显示器上所花费的时间都是延迟  

### FPS 
+ FPS: 通常指的是应用程序平均每秒运行多少帧, 因为应用程序帧率波动很正常, 所以一般计算多帧的平均时间. 注意与显示器刷新率区分.  
+ CPU Latency: CPU 完成逻辑帧所花时间
+ CPU FPS: CPU 完成渲染逻辑帧所花时间的倒数  
+ GPU Latency: GPU 完成一帧渲染所花时间
+ GPU FPS: GPU 完成一帧渲染所花时间的倒数  

<span style='color:#4cd137'>FPS 计算的是每一次 GameLoop 的平均时间的倒数, 包含了游戏逻辑更新, 渲染帧更新等完整的一帧.</span>  
CPU FPS 和 GPU FPS 代表的是渲染流程的性能, FPS 代表的是总体性能.  

---  


[HowMonitorWorks]: https://www.intel.com/content/www/us/en/developer/articles/code-sample/sample-application-for-direct3d-12-flip-model-swap-chains.html    
[IntelSample]: https://www.intel.com/content/www/us/en/developer/articles/code-sample/sample-application-for-direct3d-12-flip-model-swap-chains.html  