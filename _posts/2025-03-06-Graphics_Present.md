---
layout: post
title: Flip Model(六) Present 机制
subtitle: SwapChain::Present 机制
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
excerpt_image: /assets/images/banners/FlipModel/blt-flip-mode-present.png
tags: Direct3D 图形学 渲染
priority: 6
---

![banner](/assets/images/banners/FlipModel/blt-flip-mode-present.png)

`SwapChain::Present` 是一个看似简单的命令, 但是 Present 操作作为 CPU GPU 与 DX 驱动的同步点, 实际上有着非常复杂的内部机制. <span style='color:#4cd137'>SwapChain::Present 内部机制很复杂, 由 DXGI 控制 CPU GPU 和显示字子系统的同步.</span>  

`SwapChain::Present` 是 CPU 端在渲染流程中的最后一个指令, 但是它本身并不是渲染指令, GPU 也没有一个与之对应的 Present 指令, 它由 SwapChain 内部机制进行管理 . 接下来会详细解释 `SwapChain::Presetn` 的行为与机制.  

> 因为并没有官方文档解释相关机制, 所以关于 Present 机制都是从 FlipModel 项目, 或者一些示例侧面推出来的.  

---  

## PresentQueue  
Present Queue 是整个渲染机制中最复杂的部分, 理解了 Present Queue 的机制基本就能理解 CPU GPU 与显示系统的同步.  

PresentQueue 中缓存了从 CPU 中提交过来的逻辑渲染帧, 但是并没有这方面的官方描述, 所以以下都是根据渲染反推出来的, 不一定正确.  
1. CPU 调用 `SwapChain::Present` 方法, 没有出错并且成功返回, 就可以认定当前渲染逻辑帧已经入列, 如果 PresentQueue 满列, 则 `SwapChain::Present` 方法会阻塞, 直到 PresentQueue 的队头出列
2. PresentQueue 中的渲染逻辑帧入列时, 并不需要提交给 GPU 渲染, 而是由 PresentQueueManager 的机制决定什么时候提交给 GPU 渲染(与 BackBuffer/FrontBuffer 和 Sync 时机有关). 
3. 所有的渲染逻辑帧都会尽可能快的提交给 GPU, 并且 PresentQueueManager 会通过内部机制(可能是 Fence)得知渲染完毕的状态.
4. 在 Sync 时机到来(和垂直同步相关)时, 如果队首的渲染逻辑帧已经被 GPU 渲染完毕, 那么 PresentQueueManager 会真正的执行 PresentToMonitor 操作, 通知显示系统更换显示内存, 并且队首出列  

---  

## Present To Monitor
Present To Monitor 机制: PresentQueue 队首出列, 将已经渲染好的帧提交给显示子系统, 用于显示器显示  

Present To Monitor 实际上不受 CPU 端直接控制, 而是由 PresentQueueManager(SwapChain 关联的 DXGI 内部机制) 在同步点执行.   

<span style='color:#4cd137'>有必要说明这个同步点, 垂直同步的同步点很好理解, 得到显示系统的 vBlank 信号, PresentQueueManager 就可以开始同步操作(Present To Monitor). </span>  
垂直同步关闭的情况下, 每一次 Present API 调用都会发生同步?   

---  

### Sync
Sync 不仅仅是 VSync:  
+ 开启垂直同步时, 首先队首的帧需要 GPU 渲染完毕, 然后等待显示器的 vBlank, 执行 Present To Monitor 
+ 关闭垂直同步时, 只要队首的帧 GPU 渲染完毕即可, 执行 Present To Monitor  

### SyncInterval  
<span style='color:#4cd137'>这个参数在 BitCopy 和 FlipModel 下有不同的效果, 这里只说明 FlipModel 的效果</span>  

当 SyncInterval 为 0 时, `SwapChain::Present(0,0)` 会取消前一个 PresentQueue 中处于前列帧的剩余时间, 并且当一个新的 Frame 入列时, 自身会被丢弃. (并不是即时丢弃, 而是同步时)  

> FlipModel 中设置 SyncInterval 为 0, 并不是传统意义的 vSync Off, 还需要加上 DXGI_PRESENT_ALLOW_TEARING
  
官方文档给的例子: 当 PresentQueue 从旧到新的缓存了 A(3) B(0) C(0) D(1) E(0) 四个缓存帧, 然后再次调用 Present(0,0).  

1. 假设在同一次 vBlank 期间调用了五次 Present, 并且PresentQueue 满载容量无限
   1. A 帧, Present(3,0)
   2. B 帧, Present(0,0)
   3. C 帧, Present(0,0)
   4. D 帧, Present(1,0)
   5. E 帧, Present(0,0)
   6. F 帧, Present(0,0)
2. 再次调用 Present(), 成功入列 F 帧, 这次调用并不重要, 因为现存的最新的一次 Present 也就是 E 帧, SyncInterval 为 0, 一旦有新的 Frame 入列, E 帧注定被丢弃
3. 等到 vBlank, PresentQueueManager 对于 PresentQueue 进入出列处理
   1. 第一次 vBlank, 此时队首是 A 帧: A 帧 syncInterval 是3, A 帧出列, Present To Monitor, 
   2. 第二次 vBlank, 此时队首是 B 帧: B 帧 syncInterval 是0, 因此会取消之前 Present 的剩余时间, B 帧出列, 因为本身也可以被取消, 所以继续检查队首是否存在, 循环判定 直到 D 帧
   3. 因为 D 帧的 syncInterval 是 1, 所以不需要再次检查队首, 只需要出列, 并且 Present to Monitor
   4. 第三次 vBlank, 此时队首是 E 帧, 但实际上 D 帧的周期已经到了, 所以一定会执行 Present to Monitor, 同样循环执行 Present 操作, 直到 syncinterval > 1 或者 PresentQueue 为空 

所以呈现的效果就是:  
A 帧在 第一次 vBlank 显示, 并且只显示一帧  
D 帧在 第二次 vBlank 显示, 也显示一帧  
F 帧在 第三次 vBlank 显示  

<span style='color:#4cd137'>B, C, E 帧三个帧都属于 Dropped Frame</span>  

<span style='color:#4cd137'>传统的垂直同步不需要等待 vBlank, 一旦队首的帧渲染完毕立即执行 Present To Monitor. </span>  

---  

## 机制总结
1. 应用程序调用 `SwapChain::Present` 方法, 等待加入 PresentQueue 后, 方法返回. PresentQueue 实际上是渲染逻辑帧队列, 并且不需要 GPU 完成渲染 , `SwapChain` 关联的内部机制接管这个逻辑帧, 暂且命名为 <span style='color:#4cd137'>PresentQueueManager</span>  
   1. 如果 PresentQueue 未满载, 那么 `SwapChain::Present` 方法立即返回. 所以, 可以设置 MaximumFrameLatency 来缓存帧, 在延迟和帧平滑之间平衡. 
   2. 如果 PresentQueue 满载, 那么 `SwapChain::Present` 会阻塞, 等待 PresentQueue 出列一个元素后, 入列当前逻辑帧后返回(WaitForPresentEvent).
   3. `SwapChain::Present` 方法返回后, 对于应用程序来说, BackBuffer 与 FrontBuffer 已经轮换, CPU 可以使用新的 BackBuffer 生成渲染指令, <span style='color:#4cd137'>实际上这里都不能称为 BackBuffer, 而应该是 NextBuffer</span>. 所以  
   4. 对于应用程序来说, SwapChain 背后的机制都是异步的, 唯一的同步点就是, `SwapChain::Present` 提交渲染逻辑帧到 PresentQueue.
2. 加入 PresentQueue 后, PresentQueueManager 需要找到合适的时机将渲染指令提交给 GPU  
   1. GPU 只是一个无情的指令执行机器, GPU 本身并没有 SwapChain 这些数据结构与功能, 对于它来说, SwapChain 的 Buffer, 只是显存中的一块区域, 它只需要根据执行渲染指令并且输出到渲染流水线中的 RenderTarget 
   2. 因此, 一旦渲染逻辑帧进入了 PresentQueue, PresentQueueManager 一定会最快的速度将指令提交给 GPU. 但是, 虽然 BackBuffer 和 FrontBuffer 对于 GPU 来说没有区别. PresentQueueManager 不会提交对于 RenderTarget 为 FrontBuffer 的渲染逻辑帧  
   3. 很简单的道理, 显示器正在显示 FrontBuffer 的内容, 如果 GPU 渲染覆盖, 就会产生画面撕裂. 所以, PresentQueueManager 会锁住对 FrontBuffer 的渲染.
3. 当 GPU 执行完毕渲染逻辑帧, PresentQueueManager 会得到通知(应该也是 Fence 机制), 此时这个渲染逻辑帧对应的 BackBuffer 才是真正的准备好了.
4. 同步时机到来后, PresentQueueManager 执行出列操作, 轮换 Buffer.
   1. 垂直同步开启时, 同步时机就是 vBlank 时
   2. 垂直同步关闭时, 同步时机就是 GPU 渲染完成, BackBuffer 准备完毕时

---  

## MaximumFrameLatency
MaximumFrameLatency 只有在开启了 WaitableObject 机制时才会生效, 否则默认为3  
这个是核心机制之一, 也就是 PresentQueue 的最大容量.  

假设开启了 WaitableObject, 并且设置 MaximumFrameLatency 为 5, 并且 CPU与GPU渲染速率足够在一次 vSync Interval 内渲染大于1帧, 那么理论上 PresentQueue 容量也是5, 现在让情况简单一点, 假设 CPU 和 GPU 足够在一次 vSync Interval 中渲染 6 帧  
1. Frame1 时, 调用 `SwapChain::Present` 立即返回, 直到调用5次
2. Frame1 时, 调用第六次 `SwapChain::Present`, 此时 PresentQueue 已经满列, `SwapChain::Present` 方法阻塞(WaitForPresent)  
3. Vsync1 时, PresentQueueManager 执行出列操作, 显示器显示 Frame1 的渲染内容. 
4. 现在第六次 Present 入列, `SwapChain::Present` 方法返回, 现在 PresentQueue 中都是 Frame1 内容
5. 之后因为垂直同步, FrameN 时, 出列一帧, 入列一帧. 也就是永远延迟 5 帧  

开启垂直同步机制时, 并且 CPU 和 GPU 渲染性能足够的时候, PresentQueue 中永远有 5 帧.  

MaximumFrameLatency 牺牲了延迟来保证帧率发生波动时显示更为平滑.   

这里给出一个结论: <span style='color:#4cd137'>MaximumFrameLatency 决定了 PresentQueue 中的最大容量, 在 PresentQueue 未满列之前, 调用 `SwapChain::Presenet` 方法肯定不会被阻塞</span>    

---  


[HowMonitorWorks]: https://www.intel.com/content/www/us/en/developer/articles/code-sample/sample-application-for-direct3d-12-flip-model-swap-chains.html    
[IntelSample]: https://www.intel.com/content/www/us/en/developer/articles/code-sample/sample-application-for-direct3d-12-flip-model-swap-chains.html  