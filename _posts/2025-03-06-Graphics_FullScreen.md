---
layout: post
title: Flip Model(四) 全屏与无边框窗口
subtitle: 全屏与无边框窗口的对比
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
excerpt_image: /assets/images/banners/FlipModel/TrueFullScreen.jpg
tags: Direct3D 图形学 渲染
priority: 4
---

![banner](/assets/images/banners/FlipModel/TrueFullScreen.jpg)  


全屏与无边框窗口化也是游戏常见设置, 两者的不同和 windows 操作系统的 DWM(Desktop window manager) 相关.  

---  

## 不同点
因为 Windows 系统中, 还存在一个 DWM (Desktop Window Manager), 所有有着渲染输出的程序都会输出给 DWM, DWM 会组合这些输出, 最终输出到桌面上. 想象一下桌面多开程序, 每个程序都有自己的输出, 最终组成桌面上的画面.  

因此, 程序提交的自己的缓冲区输出, 实际上提交给了 DWM, DWM 接收后, 在当前帧进行融合, 下一帧才会将组合结果提交给显示器. 也就是中间至少多了一帧延迟.  

[使用 PresentMon 可以看到窗口程序的工作模式][TrueFullScreen]  

---  

## 设置全屏独占模式
全屏独占模式也可以称为真全屏模式, 直接向显示器输出, 不需要 DWM 这个中间商.  
在 win32 程序中, <span style='color:#4cd137'>首先要明确一点, 窗口程序的大小与工作区域大小是不同的.</span>  

因为, 全屏模式无法手动调整窗口大小, 所以, 设置全屏模式时, 就已经确定了屏幕的分辨率, 例如游戏设置里面的 1920x1080 全屏  

全屏独占模式流程重点:  
1. 设置窗口程序的风格为 WS_POPUP
2. 计算窗口工作区域大小, 全屏模式下, 工作区域大小不需要和显示器分辨率一致, 可以在 SwapChain 中设置缩放参数
  
早期 windows 系统中只有全屏模式和窗口化模式, 对应的 win32 程序对于窗口应用程序的设置参数古老且复杂. 通过设置窗口样式, 系统会判断窗口程序是否是全屏程序  

---  

## 设置无边框全屏窗口
无边框全屏窗口一定不要有 WS_POPUP, 然后保证窗口大小等于显示器分辨率, 这样系统自动识别? 并且 windows10 之后也会自动全屏优化? 这一块的机制还是挺模糊的  
具体表现在, FlipModel 程序中, 非全屏模式时, PresentLatency 会明显的多出两帧延迟, 这是因为窗口程序需要提交缓冲区给 DWM, 由 DWM 组合所有桌面窗口的渲染结果, 才能提交给显示器, 多出来的延迟就是 DWM 的操作延迟.  

而 windows 操作系统针对无边框全屏窗口会自动优化成和全屏一样的提交模式, 让无边框全屏窗口拥有和全屏一样的性能.  

> 微软官方 D3D Demo 中有专门设置全屏的[例子][SetFullScreen], 全屏设置好像和 win32 联系更加紧密  

---  

[TrueFullScreen]: https://zhuanlan.zhihu.com/p/537070602    