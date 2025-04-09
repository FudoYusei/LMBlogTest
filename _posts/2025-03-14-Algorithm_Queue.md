---
layout: post
title: Algorithm (五) 队列
subtitle: 按照某种规则排队, 只有队首能够出列
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
priority: 5
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 队列
栈在算法中的特性:  
1. 先进先出, 和栈类似, 也是简单的特性  

栈和队列本质上限制元素从容器中加入和退出. 队列可以类比成现实生活中的排队, 只能在队尾添加, 队首出  

## BFS
队列最常见的用法就是 BFS 中, 使用队列存储每一层的邻接节点, 广度优先搜索通常用于查找最短路径. 顺便提一下, DFS 更加适合验证两点之间是否存在路径  

BFS 查找最短路径的伪代码:  

```csharp
void BFS(int[] neighbors)
{
    Queue q = new Queue();
    int[] visited = new int[neighbors.Length];

    while(!q.Empty)
    {
        int levelNum = q.Count();

        for(int i = 0; i < levelNum;i++)
        {
            // 队首出列
            int cur = q.Dequeue();

            // 遍历邻接节点
            for(int j = 0; j < neighbors[cur].Length; j++)
            {
                // 如果邻接节点被访问过
                if(visited[j])
                {
                    continue;
                }

                // 未被访问过, 加入队列
                q.Push(j);
            }
            // 标记当前节点
            visited[cur] = true;
            
        }
    }
}
```  
> BFS 之所以能够在寻找最短路径中使用,实际上是因为局部最短路径也是全局最短路径
> BFS 实际上就是相邻点之间全为 1 的 Dijkstra 特例. 

## 优先队列  
优先队列是队列先进先出形式的另一种实现方式: 按照优先级排队, 优先级最高的在队首. <span style='color:#e84118'>优先级最低的并不一定在队尾</span>  

优先队列最常见的实现方式就是堆, 优先级最高在队首的称为大根堆, 优先级最低在队首的称为小根堆  