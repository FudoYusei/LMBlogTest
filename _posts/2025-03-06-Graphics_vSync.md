---
layout: post
title: Flip Model(三) 垂直同步
subtitle: vSync And Tearing
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
excerpt_image: /assets/images/banners/FlipModel/AllowTearing.jpg
tags: Direct3D 图形学 渲染
priority: 3
---

![banner](/assets/images/banners/FlipModel/AllowTearing.jpg)  

垂直同步作为游戏程序中最常见且通用的的设置, 带来的三个疑问:   
1. 开启垂直同步和关闭垂直同步有什么表现, 垂直同步的本质是什么?
2. 什么是画面撕裂?为什么会产生画面撕裂?
3. 什么是双缓冲三缓冲, 为什么需要多缓冲区?  

---  

## 显示画面的原理  
垂直同步和画面撕裂都是显示器显示的一种现象, 因此需要先理解显示画面的简单原理: 早年显示器使用一个扫描枪, 就像人写字一样, 一行一行的垂直扫描, 从顶部扫描到底部就完成一帧画面的绘制.  

显示器的刷新率, 是显示器本身的属性, 表示显示器在 1s 内会刷新多少次. 视为显示器执行一个定时程序, 每隔一定时间读取一次缓冲区, 将缓冲区的内容显示到屏幕上.  

显示器的周期可以粗略分为 vBlank 和 显示画面所需的时间. vBlank 期间进行同步, vBlank 结束, 显示器开始使用缓冲区内容进行行扫描刷新画面.  

---  

## vBlank 与 vSync
扫描枪从显示器底部重置回顶部的时间, 称为 vBlank, 这时是一帧结束还未进行下一帧绘制的空闲时间. 这段时间显示器不会显示新的内容, 利用这段时间, 应用程序与显示器进行帧率同步, 这个机制被称为 vSync.  

垂直同步最通用的一个解释就是, 让游戏程序的帧率与显示器刷新率一致, 优点是避免画面撕裂, 缺点是操作延迟, 高竞技性游戏都是推荐关闭. 关闭垂直同步, 提高游戏帧率, 降低操作延迟, 但是会产生画面撕裂.  

从整体上来说, vSync 的本质就是应用程序与显示器的同步操作. 应用程序渲染完毕的帧需要等待显示器的 vBlank 才能进行提交, vBlank 结束,  显示器才会刷新当前画面, 显示应用程序提交的内容.  
关闭垂直同步后, 应用程序渲染完毕的帧无需等待可以立即提交, 当提交操作正好处于显示器刷新画面的期间, 例如, 行扫描进行到一半, 显示内容被替换, 那么显示器上下两半的内容不一致, 就是画面撕裂.  

---  

## vSync And Tearing
在 FlipModel 中, vSync 就是 `SwapChain::Present` 的一个参数: SyncInterval  
表示提交的帧最多能够在显示器上存在的垂直同步周期. 设置 SyncInterval 为 0 时, 表示该提交帧随时可以被刷新, 也就是关闭垂直同步. 在 FlipModel 和非 FlipModel 有着不同的解释, 详见[官方文档][PresentAPI]  

<span style='color:#4cd137'>对比两者的解释, 以前版本 SyncInterval 指的是延迟提交, 现在版本指的是持续时间, 两者的差异暗中关联了画面撕裂的机制.</span>    

画面撕裂本质上是在 vBlank 之外进行了额外的 Present To Monitor 操作.  <span style='color:#4cd137'>只要保证只在 vBlank 时, 执行一次 Present To Monitor, 那就一定不会有画面撕裂. </span>  

在新版的 FlipModel 中, 垂直同步与画面撕裂并不等同, 两者是分开的选项. FlipModel 后续添加了一个 [DXGI_PRESENT_ALLOW_TEARING][VsyncOff], 实现了传统意义上的垂直同步关闭功能.  

简单解释一下 FlipModel 默认的提交机制: 只在 vBlank 时执行一次 Present To Monitor, 此时会根据队首帧的 SyncInterval 参数进行处理. 一直与显示器同步, 也就不会产生画面撕裂.  
新添加 DXGI_PRESENT_ALLOW_TEARING 参数, 就是将 FlipModel 还原成旧的提交机制. SyncInterval 为 0 时, 立即执行 Present To Monitor  

---  

## 缓冲区
<span style='color:#4cd137'>开启垂直同步还有一个隐藏特性, 那就是为了防止画面撕裂, 显示器正在使用的缓冲区不被允许写入(并不是显示器保证, 而是 DXGI 的内部机制保证的).</span>  

### 单缓冲区
假如只有一个缓冲区, 并且开启垂直同步, 渲染流程如下:  
1. 应用程序渲染好一帧 
2. 等到 vBlank, 将渲染好的帧提交给显示器
3. 显示器显示该帧内容, 此时, 应用程序无法继续新的一帧的渲染, 因为显示器正在显示缓冲区中的内容, 应用程序写入新的内容会导致画面撕裂.  

单缓冲区实际上无法保证垂直同步, 因为无法保证画面撕裂.   

### 多缓冲区
双缓冲区, 并且开启垂直同步, 渲染流程如下:  
1. 应用程序渲染好一帧, 存储在 BufferA 中
2. 显示器进入 vBlank 状态, 
3. SwapChain 内部驱动提交 BufferA 给显示器, 并且SwapChain 轮换后台缓冲区(后台缓冲区变为前台提交给显示器, 原前台缓冲区轮换为后台)
4. 显示器退出 vBlank 状态, 开始显示驱动提交过来的缓冲区内容, 同时应用程序并行的开始使用 SwapChain 当前的后台缓冲区渲染下一帧

双缓冲区保证应用程序在等待显示器同步的时候, 可以在显示器显示一帧的同时, 渲染下一帧.  
<span style='color:#fbc531'>双缓冲有一个隐含的问题, 天然具有延迟性</span>  
极端例子, 显示器刷新率 10s 1帧  
10s, 第 1 帧同步后, 应用程序立马渲染下一帧, 这一帧实际上是基于 10s 的逻辑帧.  
但是这个逻辑帧是作为第 2 帧同步提交给显示器显示的, 所以 20s 第 2 帧同步后, 显示器显示的是第 10s 的内容  
也就是双缓冲区天然落后一帧.  

三缓冲区, 为了解决双缓冲区天然落后一帧的提出来的机制, 但是好像现在游戏都没有使用, 可能是大部分配置达不到显示器刷新率的三倍  
1. 应用程序渲染好一帧, 存储在 BufferA 中
2. 显示器进入 vBlank 状态, 
3. SwapChain 内部驱动提交 BufferA 给显示器, 并且SwapChain 轮换后台缓冲区(后台缓冲区变为前台提交给显示器, 原前台缓冲区轮换为后台)
4. 显示器退出 vBlank 状态, 开始显示驱动提交过来的缓冲区内容, 同时应用程序并行的开始使用 SwapChain 当前的后台缓冲区渲染下一帧
5. 与双缓冲不同的是, 后台渲染完毕后, 并不会等待下一次 vBlank, 而是继续渲染, 直到 vBlank 时, 提交最新的那一帧  

这样, 即保证了不会出现画面撕裂, 还可以得到低延迟, 缺点就是占用三倍的渲染资源.  
  
> 现在大部分游戏的三缓冲只是单纯在增加一个后台缓冲, 这样可以让 GPU 预先渲染两帧, 更好的容纳帧率波动, 缺点是更大的延迟  

---  

[PresentAPI]: https://learn.microsoft.com/zh-cn/windows/win32/api/dxgi/nf-dxgi-idxgiswapchain-present    
[VsyncOff]: https://learn.microsoft.com/zh-cn/windows/win32/direct3ddxgi/variable-refresh-rate-displays