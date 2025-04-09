---
layout: post
title: Algorithm - 寻路算法
subtitle: Dijkstra A*
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

寻找最短路径  

## Dijkstra 算法
Dijkstra 算法并不是贪心算法, 贪心算法的缺点在于有时候局部最优解无法推导出全局最优解. 而 Dijkstra 每一步都是全局最短路径.  


## A* 算法
启发式算法, 使用权值提高 Dijkstra 算法的效率.  