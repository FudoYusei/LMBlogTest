---
layout: post
title: Flip Model(七) Present Statistics
subtitle: 解析 Present Statistics 数据结构
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
excerpt_image: /assets/images/banners/FlipModel/frame-sync-glitch-recover.png
tags: Direct3D 图形学 渲染
priority: 7
---

![banner](/assets/images/banners/FlipModel/frame-sync-glitch-recover.png)

`DXGI_FRAME_STATISTICS` 是 DXGI 围绕着 PresentQueue 的[统计数据结构][DXGI_FRAME_STATISTICS] .针对 PresentQueue 的统计自然也是深度绑定 PresentQueue 的机制.  

> 官方文档的描述只能说如讲, 奇怪的是, 网上几乎也搜不到关于这个数据结构的任何疑问或者详细的解释  

---  

## 参数  
DXGI_FRAME_STATISTICS 数据结构本身倒是不复杂, 只有四个数据成员:  
+ PresentCount 自程序开始(SwapChain创建开始) Present To Monitor 的次数(并不严格等于调用 `SwapChain::Present`的次数)  
+ PresentRefreshCount 上一次 Present To Monitor 操作时, 显示器从程序启动时(交换链创建时) vBlank 的总数
+ SyncRefreshCount 上一次 DXGI 进行同步操作时, 显示器从程序启动时(交换链创建时) vBlank 的总数  
+ SyncQpcTime 上一次 DXGI 进行同步操作时, 记录的高精度计数 
+ SyncGPUTime 保留(无用)

| 成员                  | 核心意义                                                                                             | 调试中的应用场景                                                                                                               |
| --------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `PresentCount`        | 自程序开始(SwapChain创建开始) Present To Monitor 的次数(并不严格等于调用 `SwapChain::Present`的次数) | 垂直同步开启时, CPU 生成帧并会预设一个 PresentID, 当 `SwapChain::PresentCount` 等于 PresentID时, 表示当前帧 Present To Monitor |
| `PresentRefreshCount` | 上一次 Present To Monitor 操作时, 显示器从程序启动时(交换链创建时) vBlank 的总数                     | 可以得知当前提交帧的 vBlank 数, 与 SyncRefreshCount 结合使用, 可以检测是否有 Glitch 发生                                       |
| `SyncRefreshCount`    | 上一次同步操作时经过的 vBlank 数量                                                                   | 当开启垂直同步时, 等于 vBlank 数量, 与 PresentRefreshCount 结合使用, 可以检测是否有 Glitch 发生                                |
| `SyncQPCTime`         | 上一次同步操作时的 CPU 时间戳。                                                                      | 开启垂直同步时, 就是 vSync 发生的时间点                                                                                        |
| `SyncGPUTime`         | 最后一次垂直同步的 GPU 时间戳（若支持）。                                                            | 保留, 现在无效, 值为 0。                                                                                                       |

---  

## PresentCount  
因为 PresentQueue 可以缓存 CPU 端的提交帧, 所以调用 `SwapChain::Present` 只是入列, 并没有触发 Present To Monitor 操作, PresentCount 也就不会增加.  

> 根据 DropFrame 的代码来看, 只要进入 PresentQueue 中的帧, 哪怕没有显示在显示器上, 只要通过 Present To Monitor 出列, 都会令 PresentCount 加 1. <span style='color:#4cd137'>所以只有 Present Queue 中出列的渲染逻辑帧会让 PresentCount 加 1, 因此 PresentCount 并不严格等于调用 Present API 的次数</span>  

---  

## PresentRefreshCount
自从上一次 Present To Monitor 操作时, 程序经过的 vBlank 次数. 官方文档提到过一句话: 如果每帧都进行一次 Present, 那么 PresentRefreshCount 等于 SyncRefreshCount  

---  

## SyncRefreshCount  
SyncRefreshCount 可以从官方给的 [DetectGlitch](DetectGlitch) 示例中反推出来: 自从上一次同步操作以来, 程序(SwapChain创建时)经过的 vBlank 次数.  

SyncRefreshCount 和 PresentRefreshCount 的区别在哪呢? 从字面意义上看, Sync 显然不等同于 Present  
同步点并不一定执行 Present To Monitor. 例如, 在开启垂直同步的情况下, Sync 指的是显示器的周期 vBlank:  

1. 因为开启垂直同步, 显示器的 vBlank 就是同步点, 每经过一次 vBlank, SyncRefreshCount 增加 1, SyncRefreshCount 等于 vBlank 数
2. CPU 和 GPU 渲染性能足够, 每一次 vBlank, PresentQueue 都可以进行一次(或者多次) PresentToMonitor. 那么 PresentRefreshCount 等于 SyncRefreshCount
3. 如果 CPU 和 GPU 渲染性能不够, 在某一次 vBlank, PresentQueue 处于空列状态, 此时肯定不会执行 PresentToMonitor, 因为上一次的 Present 没变, PresentRefreshCount 就不会改变.  

---  

## DetectGlitch
[DetectGlitch 官方示例][DetectGlitch] , Glitch 发生在使用垂直同步时, 显示器显示的画面已经落后超出必要延迟(缓冲区造成的延迟)   
1. 在调用 `SwapChain::Present` 成功返回后, 记录该帧的 PresentID(`GetLastPresentCount`), 并计算出该帧预计的 PresentRefreshCount(作为 ExpectedSync). 
2. 在某个时间点检索 DXGI_FRAME_STATISTIC 信息. 得到 PresentCount 和 SyncRefreshCount(作为 ActualSync) 
3. 通过当前 PresentCount 匹配记录帧的 PresentID. 对比 SyncRefreshCount 和记录帧中预计的 PresentRefreshCount  
4. 如果当前 SyncRefreshCount(ActualSync) 大于 PresentRefreshCount(ExpectedSync), 说明该帧在显示器上显示的时机迟于理想情况, 必然发生了卡顿.  
5. Glitch Count 等于 ActualSync - ExpectedSync, 想要消除 Glitch Count, 需要 "Drop" 对应的帧数, 但是 PresentQueue 中的帧不会被直接丢弃, 而是丢弃新的帧: 对应数量的新的帧 Present(0,0). 执行到这些帧时, 会直接被丢弃.

<span style='color:#fbc531'>为什么要消除 Glitch? 好像不消除也没事, 因为应用程序生成帧是根据现实时间来的, 不可能根据 vSync 来吧</span>  

> 延迟的帧数 N = PresentQueueDelay(5) + GlitchCount(3)  

---  

[DXGI_FRAME_STATISTICS]: https://learn.microsoft.com/zh-cn/windows/win32/api/dxgi/ns-dxgi-dxgi_frame_statistics    
[DetectGlitch]: https://github.com/MicrosoftDocs/win32/blob/docs/desktop-src/direct3ddxgi/dxgi-flip-model.md#avoiding-detecting-and-recovering-from-glitches  