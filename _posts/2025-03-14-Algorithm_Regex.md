---
layout: post
title: Algorithm - 正则表达式
subtitle: 
author: LM
categories: 算法
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
excerpt_image: /assets/images/Algorithm/Algorithm_Overview.png
tags: 算法
priority: 20
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 正则表达式
正则表达式在文本处理应用的非常广泛, 因此值得单独拿出来讲一讲. 正则表达式有着看上去简单但是组合起来非常复杂的规则.  
我们学习的基础算法也能实现一个简单的正则解析  

## 正则表达式匹配
[正则表达式匹配](https://leetcode.cn/problems/regular-expression-matching/solutions/295977/zheng-ze-biao-da-shi-pi-pei-by-leetcode-solution/)  

### 有限状态机
正则表达式的规则可以解析成有限状态机, 然后使用 DFS 来匹配路径.  
关键问题就是构建一个合适的[有限状态机](https://zhuanlan.zhihu.com/p/104291251).  

可以类比常见的在二维网格中:  
1. 每一个格子就是状态
2. 每一个格子前往下一个格子的操作就是 Pattern 的匹配
3. 网格就是 Pattern 形成的有限状态机.   

该题的正则规则很简单:  
1. 字母直接匹配
2. . 表示匹配一个任意字符
3. * 表示重复匹配零次或者多次

正则表达式的匹配过程类似于 WordSearchII:  
1. 匹配方式根据 PatternNode 类型匹配
2. 操作路径也是根据 PatternNode 类型进行  

<span style='color:#4cd137'>关键点在于 * 怎么处理?</span> 本题的状态机很简单:  
1. . 影响匹配条件
2. * 影响路径

```csharp

```  
