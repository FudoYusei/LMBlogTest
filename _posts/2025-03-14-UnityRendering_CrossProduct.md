---
layout: post
title: UnityRendering (二) 叉乘
subtitle: 叉乘到底是什么
author: LM
categories: UnityRendering
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_CrossProduct.png
tags: UnityRendering
priority: 2
---

![banner](/assets/images/UnityRendering/UnityRendering_CrossProduct.png)  

叉乘是图形渲染中很常见的符号操作, 叉乘在立体几何中用于求解法向量, 坐标系手性决定了叉乘的规则.  

---  

## 偏手性
叉乘最令人困惑的地方就是在左右手坐标系中不同的表现. <span style='color:#4cd137'>左右手坐标系的转换可以类比于现实中的镜像, 不仅仅是位置改变, 方向也会被改变.</span> 左右手坐标系不仅仅顶点坐标会被改变, 顶点绕序也会改变, 法向量方向也会改变.  
  
---  

### 实例一 顶点坐标未改变
左手坐标系基向量 $ \vec{i} \vec{j} \vec{k} $ 中三个顶点 A(0, 0, 0) B(1, 0, 0) C(0, 1, 0) 形成一个三角面, 三角面的法向量为 $ \vec{AB} \times \vec{AC} = (0, 0, 1) $, 也就是 $ \vec{k} $  

现在将三个顶点映射到右手坐标系, 右手坐标系基向量 $ \vec{l} \vec{m} \vec{n} $, 其中 $ \vec{n} = -\vec{k} $. 映射点 $ A^{'}(0, 0, 0) B^{'}(1, 0, 0) C^{'}(0, 1, 0) $, 可以看出顶点坐标没有变化, 叉乘结果依然是 (0, 0, 1). 但是对应的基向量为 $ \vec{n} $, 也就是 $ -\vec{k} $, 法向量方向发生了改变.  

### 实例二 顶点坐标改变
同样左手坐标系中三个顶点 A(0, 0, 0) B(0, 0, 1) C(1, 0, 0) 叉乘结果为 (1, 0, 0)  
转换到右手坐标系中三个顶点 $ A^{'}(0, 0, 0) B^{'}(0, 0, -1) C^{'}(1, 0, 0) $ 叉乘结果为 (-1, 0, 0). 对应的基向量 $ \vec{l} = \vec{i} $, 所以叉乘结果依然相反.  

### 实例三 偏手性的判断
<span style='color:#4cd137'>典型的判断偏手性的方法: 基于当前坐标系的手性, 例如, 当前坐标系是左手坐标系, 已知三个向量组成一个空间, 判断这三个向量手性的方法, 就是按照顺序取出前两个基向量, 得到叉乘结果, 再与第三个向量点乘获取两者的方向对比, 同向则手性相同, 反向则手性不同. 因为这三个基向量的运算规则与左手坐标系相反.</span>  
  

### 实例四 偏手性的使用
为顶点计算切向量空间的过程中, 首先获得切向量空间的三个基向量在对象空间中坐标表示, 然后计算切向量空间相对于对象空间的偏手性. <span style='color:#4cd137'>接下来为了节省存储空间, 不会保存所有的向量, 顶点法向量本来就有, 两个切向量, 只需要保存一个, 另一个通过叉积计算得到.</span> 如果顶点空间与对象空间偏手性不一致, 叉乘得到的结果还需要取反, 也就是记录偏手性的 w 值为 -1.  

### 实例五 顶点绕序
顶点绕序只有两种: 顺时针和逆时针, 分别对应左手坐标系和右手坐标系. 当我们说左手坐标系, 正面顶点绕序是顺时针, 右手坐标系, 正面顶点绕序是逆时针.  

实际上包含隐藏的前置条件: 观察者处于 +Z 轴, 向原点看去, X -> Y 轴的顺序就是顶点绕序. 也就是该手性坐标系叉乘运算的规则.   

---  

## 总结

手性转变会改变叉乘规则, 等同于改变顶点绕序. <span style='color:#4cd137'>但是, 在标准渲染流程中, 世界坐标系到屏幕空间, 并没有改变, 因此我们设置正面顶点绕序来告知屏幕空间在世界空间中的叉乘规则(正面顶点绕序), 这样就能够计算出正确的法向量</span>  
因此, 根据世界坐标系的手性设置正面顶点绕序, 观察者就可以根据这个顶点绕序判断正反面.  

---  

