---
layout: post
title: Graphics Rendering (五) 手动实现软渲染-动态相机
subtitle: 移动的摄像机
author: LM
categories: 图形渲染
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
tags: 图形渲染
priority: 5
---

![banner](/assets/images/banners/ComputerGraphics/Projection.png)  

TinyRenderer 项目本身没有使用视锥体来构建 Camera Space, 简单的将 Camera 当作透视关系的光线聚集点. 并且与标准流程的视锥体映射近平面不同, 直接在世界坐标系中构建了映射关系.  

---  

## 矩阵变换
矩阵变换坐标的本质是将顶点从从一个基坐标空间转换到另一个基坐标空间中, 顶点的位置不变, 只是坐标改变.  

首先, 要记住三维空间中的线性变换就是矩阵, 矩阵就是[线性变换](https://blog.csdn.net/Xuuuuuuuuuuu/article/details/120488971). 之后加上平移操作, 被称为仿射变换.  

线性变换满足的基础数学定律:  
$$ M \vec{v} = T(\vec{v}) = T(v_x \vec{e_1} + v_y \vec{e_2} + v_z \vec{e_3}) \\
= v_x T(\vec{e_1}) + v_y T(\vec{e_2}) + v_z T(\vec{e_3})
= \left[
    \begin{matrix}
        T(\vec{e_1}) & T(\vec{e_2}) & T(\vec{e_3})
    \end{matrix}
\right]
\left[
    \begin{matrix}
        v_x \\ v_y \\ v_z
    \end{matrix}
\right]
$$  

因此, 想要得到对一个空间进行线性变换. 只要得到源空间的基向量变换到目的空间的基向量, 就可以组成对应的转换矩阵. <span style='color:#4cd137'>对基向量施加的映射关系组成了空间映射矩阵</span>:  
$$ \vec{v}\_{world} = M\_{objectToworld}\vec{v}\_{object} $$  


已知对象空间的坐标, 使用线性变换, 将坐标变换到世界坐标系中指定的位置.  
我们要求出$ M_{world} $, 只要得到对象的基向量在世界坐标转换后的向量即可.  
这里, 对象空间基向量为 $ \vec{i} \vec{j} \vec{k} $  

<span style='color:#4cd137'>注意区别: 世界坐标系中的同一个顶点转换成view space坐标系表示, 也就是顶点没有变换, 换了基向量.</span>  


针对世界空间转换到View Space中:  
$$ M\_{view} \vec{v}\_{view} = M\_{world} \vec{v}\_{world}  $$  
$$ \vec{v}\_{view} = M\_{view}^{-1} M\_{world} \vec{v}\_{world}  $$  
只要二者的变换矩阵都是基于同一个基向量即可直接相乘变换.  

---  

## view space  
这里我们需要求出view space的逆转矩阵, 齐次4D矩阵的逆矩阵比较特殊, 可以使用分块矩阵快速求出  

$$ F^{-1} = 
\left[
    \begin{matrix}
    M^{-1} & -M^{-1}T \\
    0 & 1
    \end{matrix}
\right] $$  

```
// eye camera的位置, center指向的对象物体中心, up 摄像机向上的向量
mat<4,4> lookat(vec3 eye, vec3 center, vec3 up)
{
    vec3 z = (eye - center).normalize();
    vec3 x = cross(up, z).normalize();
    vec3 y = cross(z,x).normalize();
    mat<4,4> mInverse = mat::identity();
    mat<4,4> translationReverse = mat::identity();

    for(int i = 0; i < 3; i++)
    {
        // 正交矩阵的逆转矩阵就是转置矩阵
        mInverse[0][i] = x[i];
        mInverse[1][i] = y[i];
        mInverse[2][i] = z[i]

        translationReverse[i] = -eye[i];
    }
    return mInverse * translationReverse;
}
```  
$ mInverse = 
\left[
\begin{matrix}
    M^{-1} & -M^{-1}T \\
    0 & 1
\end{matrix}
\right]
\left[
\begin{matrix}
    M & T \\
    0 & 1
\end{matrix}
\right] = I$  

---  

## viewport  
当我们将所有坐标转换到NDC坐标系后.  
经过透视除法, 会获得[-1,1]范围内的xy坐标.  
此时, 我们需要将xy坐标映射到屏幕上的一块矩形区域  
矩形区域由四个参数决定:  
(x, y, w, h)  
$ ScreenX = (P_x + 1) * (w/2) + x $

因此可以构造出viewport矩阵  
$ \left[
    \begin{matrix}
    \frac{w}{2} & 0 & 0 &x + \frac{w}{2} \\ 
    0 & \frac{h}{2} & 0 & y + \frac{h}{2} \\
    0 & 0 & 1 & 0 \\
    0 & 0 & 0 & 1
    \end{matrix}
\right] $  

```
mat<4,4> Viewport(int x, int y, int w, int h)
{
    mat<4,4> m = mat::identity();
    
    m[0][3] = x+w/2.f;
    m[1][3] = y+w/2.f;
    
    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    
    return m;

    return m;
}
```  

---  

## 法向量的线性变换
设 $\vec{t}$  是平面切向量  
设 $\vec{n}$ 是平面法向量  
设 M 为切向量变换矩阵  
设 F 为法向量变换矩阵  
变换后的切向量为 $M \vec{t}$  
变换后的法向量为 $F \vec{n}$  
需要求出F, 使得 $ M \vec{t} \cdot F \vec{n} = 0 \\ $ 成立  
$(M \vec{t})^t F \vec{n} = (\vec{t}^t M^T)(F \vec{n} = \vec{t^T}(M^T F) \vec{n}
$  
因为本身 $ \vec{t} \cdot \vec{n} = \vec{t}^T n = 0 $  
所以可以得出 $ M^T F = I $  
$F = (M^T)^{-1}$  

---  