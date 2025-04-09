---
layout: post
title: Unity Coroutine (二) 异步与协程
subtitle: 什么是协程
author: LM
categories: Unity3D
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
tags: Unity3D 游戏引擎
priority: 2
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine2.jpg)  

[协程的历史进程][协程的历史进程]:    
1. 最初, windows 系统使用的调度方式就是协作式进程(非抢夺式多任务), 必需等待占有 CPU 的进程主动让出执行权. 所以一个进程死锁会引起所有进程瘫痪
2. Windows95, 抢占式多任务: 每个进程一个时间片, 用完强制休眠, 抢占式多任务标志着真正支持多进程的操作系统诞生.  

协作式多任务: 进程独立运行, 每个进程自己发扬风格, 不执行规模过大的运算. 或者执行大规模运算时, 每隔一段时间发扬风格主动调用一下OS提供的特定API, 让出控制权给其他进程  
抢夺是多任务: 操作系统监控所有进程, 公平分配CPU时间片等资源给每个应用; 一个应用用完了自己的份额, 操作系统强制暂停它的执行, 保存现场, 把CPU安排给另外一个进程. 避免坏进程影响系统的正常运行.  

协程是什么? 简单来说, 在每一轮主循环时, 异步操作排队, 一个一个执行. 这样在外界看起来, 每个异步操作都同时在运行.  

## 协作式多任务的好处
梳理好同步的逻辑之后, 不需要使用抢夺式多任务, 只需要让该做的事情做完再让出主动权即可.  
减少线程切换和同步开销.  

这个做法，使得协作式多任务之间执行权的交接点极为明晰；那么只要逻辑考虑清楚了，锁就是完全没必要的——反正不会抢夺嘛，事情没告一段落我就不会交执行权；交执行权之前确保不存在“悬置的、未确定未提交的修改”，脏读脏写就杜绝了。  

协程本质上是一种在用户空间实现的协作式多线程架构.  
它不能让OS直到自己的存在, 无法利用多核CPU/GPU的多线程支持; 恰恰是它的优点.  

OS并不在系统级别支持协程, OS的底层逻辑就是抢夺式多任务.  
<font color=#4cd137>协程的缺点就是: 无法在CPU上并行</font>  

没有OS的抢夺式多任务, 意味着OS不会调度协程, 需要用户自己调度协程  

<font color=#4cd137>协程之间的具体执行顺序可能千变万化, 但是协程执行权切换只会发生在用户明确放弃执行权之后, 使用yield return表示当前工作阶段完毕, 可以让出主动权</font>  

<font color=#4cd137>协程的应用场景在于: 开点小差做点微小的工作的同时, 不希望阻塞主要执行逻辑. 这种简单的引用场景多半也没有什么数据需要共享</font>  

## 协程的正确用法
<font color=#4cd137>协程是一种抛弃了在CPU上并行执行能力的, 协作式多任务的执行框架</font>  
大部分使用场景上可以代替线程, 并且无需担心棘手的数据相关问题  
核心点在于, 主动交出控制权.  

协程和线程并不互斥.  
协程可以内部调用线程 或者线程内部, 借助协程并行IO  

不同线程间不能共享一个协程控制器, 不然等同于一个线程控制   

## 协程与多线程区别
[协程与多线程的本质区别][协程与多线程]: 多线程调度由系统负责, 协程可以由用户自定义调度器. <span style='color:#4cd137'>Unity 的协程调度器处于 UI 线程</span>  

<span style='color:#4cd137'>当对比多线程与协程时, 会有一个我们平时一直忽略的问题. 那就是线程由谁执行?协程由谁执行.</span>  
我们只需要在写出代码并点击运行, 就默认它会被执行. 当我们再创建多个线程, 也默认这些线程会自动运行. 同样在 Unity3D 中, 使用 `StartCoroutine` 方法开启一个协程, 也是默认它被运行. <span style='color:#4cd137'>而多线程和协程的本质区别就是由谁来运行</span>  

<span style='color:#4cd137'>线程的调度运行由操作系统负责, 协程的调度运行由用户自定义实现.</span>  

Unity3D 的协程就是简单的排队轮流执行, 所有的协程都是处于主线程上, 协程运行实际上就是主线程运行, 如果某个协程"卡死", 所有的协程包括主线程同样都会"卡死". 而线程具有可抢占式特性, 一个线程卡死, 其他线程依然可以抢占 CPU 时间, 互相之间不会影响.  

> 协程可以自定义调度器, 可以决定协程的运行方式, 那么将协程指定给线程也是可以的, 这种被称为跨线程协程 
  
[协程出现的原因]: https://www.zhihu.com/question/50185085/answer/183463734  
[协程与多线程]: https://blog.csdn.net/weixin_44575037/article/details/105513014    
[协程的历史进程]: https://zhuanlan.zhihu.com/p/147608872  
