---
layout: post
title: Computer Graphics (三) 手动实现软渲染-光栅化
subtitle: 光栅化三角形
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
excerpt_image: /assets/images/GraphicsRendering/scanline4.png
tags: 图形渲染
priority: 3
---

![banner](/assets/images/GraphicsRendering/scanline4.png)  

TinyRenderer 比起软渲染程序这个称呼, 更加准确的称呼是软光栅化程序. 因为主体就是实现了一个软光栅化功能, 几乎没有其他与渲染相关的功能.  

<span style='color:#4cd137'>有些理论将光栅化分为三角形设置与三角形扫描两个阶段.</span>  
+ 三角形设置, 顶点进行图元装配, 组装成三角形, 光栅化之后, 渲染单位从顶点变成了屏幕三角形
+ 三角形扫描, 通过扫描线方式填充屏幕三角形覆盖的所有像素.  

---  

## Filling Triangle
前面展示了如何绘制三角形的边, 专门绘制边的模式也有对应的称呼: wireframe mode(线框模式)  
Filling Triangle 字面意义上的填充三角形, 将屏幕空间的三角形用像素填充满.  

核心在于: 已知屏幕空间三角形三个顶点的坐标, 计算得到空间三角形覆盖的屏幕像素点, 再给这些像素点赋值三角形的颜色, 整个过程就是三角形填充. <span style='color:#4cd137'>难点在于找到三角形覆盖的像素点</span>  
  
---  

## 扫描线
现代显示器刷新画面也是垂直扫描, 光栅化也是同样从上往下垂直扫描.  

已知屏幕空间三角形三个顶点的屏幕坐标, 光栅化屏幕三角形代码如下:   
```cpp 
void triangle(vec2 t0, vec2 t2, TGAImage &image, TGAColor color)
{
    line(t0,t1,image,color);
    line(t1,t2,image,color);
    line(t2,t0,image,color);
}

// ...

Vec2i t0[3] = {Vec2i(10, 70),   Vec2i(50, 160),  Vec2i(70, 80)}; 
Vec2i t1[3] = {Vec2i(180, 50),  Vec2i(150, 1),   Vec2i(70, 180)}; 
Vec2i t2[3] = {Vec2i(180, 150), Vec2i(120, 160), Vec2i(130, 180)}; 
triangle(t0[0], t0[1], t0[2], image, red); 
triangle(t1[0], t1[1], t1[2], image, white); 
triangle(t2[0], t2[1], t2[2], image, green);
```
用于绘制三角形的代码，应该具有如下特征
+ 简单与快速
+ 具有对称性：与顶点顺序无关（那怎么分正反面？）
+ 如果两个三角形具有共同的顶点，不会因为光栅化的近似而导致两者之间有空隙  

传统的线扫描法使用如下方法：
+ 通过y轴坐标排序，从上往下扫描
+ 同时光栅化三角形左右侧
+ 再左右边界点之间绘制一个水平线段  


### 左右边界
遍历三角形上下边界之间的 y 值, 求出对应所有的左右边界: $ x_{left} $ 与 $ x_{right} $, 左右边界之间的点就是三角形覆盖的像素点.  
如何区分三角形左右侧，实质在于找出最长的那条边，最长的边必然形成一侧，另一侧就是两个较短边。进行顶点y轴排序，找到最高和最低点，这两点连线就是三角形最长边。
```csharp
if(t0.y > t1.y) std::swap(t0, t1);
if(t0.y > t2.y) std::swap(t0, t2);
if(t1.y > t2.y) std::swap(t1, t2);
line(t0, t1, image, green);
line(t1, t2, image, green);
line(t2, t0, image, red);
```  

### 光栅化
假想一个水平扫描线,从最低点向最高点扫描,每次同时光栅化左右两边界之间的所有点.
```csharp
void triangle(vec2 t0, vec2 t1, vec2 t2, TGAImage& image, TGAColor color)
{
    // orderbyY(t0, t1, t2), bubble sort
    // 根据y值排序,从小到大分别是 t0, t1, t2
    if(t0.y > t1.y) std::swap(t0, t1);
    if(t0.y > t2.y) std::swap(t0, t2);
    if(t1.y > t2.y) std::swap(t1, t2);

    // 根据相似三角形,求出两边上的点.
    int totalHeight = t2.y - t0.y;
    for(int i = 0; i < tottalHeight; i++)
    {
        // i代表水平扫描线相对于t0的高度,超过t1,或者t0和t1一样高,也就是平底.
        bool secondHalf = i > t1.y - t0.y || t1.y == t0.y;
        int segmentHeight = secondHalf ? t2.y - t1.y : t1.y - t0.y;
        float alpha = (float)i / totalHeight;
        float beta = (float)i / totalHeight;
        vec2 A = t0 + (t2 - t0) * alpha;
        vec2 B = second_half ? t1 + (t2-t1) * beta : t0 + (t1 - t0) * beta;
        if(A.x > B.x) std::swap(A, B);
        for(int j = A.x; j <= B.x; j++)
        {
            // t0.y + i != A.y !!! due to integer adjust
            image.set(j, t0.y + i, color);
        }
    }
}
```  

---  

## 包围盒
现代渲染管线中的光栅化阶段, 由 GPU 负责完成, GPU 最擅长的就是大批量并行计算. 上面这种充满了逻辑判断的方法不适用于 GPU.  

包围盒法是一种普遍使用的光栅化方法:  
1. 将三角形使用矩形包围盒覆盖, 减少遍历的像素范围.
2. 依然使用扫描线方法, 但不用计算三角形左右边界. 而是计算扫描线上的像素相对于三角形三个顶点的插值, 也成为[重心坐标](https://zhuanlan.zhihu.com/p/140907023).
3. 重心坐标中任意值小于0, 说明该像素点不在三角形内, 直接丢弃, 反之, 绘制该像素点.

伪代码:  
```c
triangle(vec2 points[3])
{
    vec2 bbox[2] = find_bounding_box(points);
    for(each pixel in the bouding box)
    {
        if(inside(points, pixel))
        {
            put_pixel(pixel);
        }
    }
}
```  

### 判断一个点是否在三角形内部
<font color=#4cd137>判断一个点是否在三角形内, 实际上是在屏幕空间中做的插值, 因此没有z坐标, 并不清楚重心坐标的公式是否能从二维空间应用到三维空间中</font>  


<font color=#00a8ff>重心坐标:</font> $P = w_1A + w_2 B + w_3 C$  
<font color=#00a8ff>重心坐标权重和为1:</font> $w_1+w_2+w_3 = 1$  
<font color=#00a8ff>两者可以互相转换：</font>
代入 $w_1 = 1 - w_2 - w_3$后,得:  
$P = A + w_2(B-A) + w_3(C-A)$  
$w1=1-u-v, w2=u, w3=v$  
<font color=#00a8ff>向量公式: </font>$P = A + \mu \overrightarrow{AB}  + \nu \overrightarrow{AC}$  


因此, 求出(u, v, 1)这个向量即可.  

<font color=#00a8ff>线性方程组:</font>
$\begin{cases}
  & \mu\overrightarrow{AB}_x + \nu\overrightarrow{AC}_x+\overrightarrow{PA}_x = 0  \\
  & \mu\overrightarrow{AB}_y + \nu\overrightarrow{AC}_y+\overrightarrow{PA}_y = 0
\end{cases}$  

<font color=#00a8ff>矩阵方程:  </font>
$
\begin{cases}
  & 
\begin{bmatrix}
u & v & 1 
\end{bmatrix}
\begin{bmatrix}
\overrightarrow{AB}_x \\
\overrightarrow{AC}_x \\
\overrightarrow{PA}_x \\
\end{bmatrix} = 0
\\
&
\begin{bmatrix}
u & v & 1 
\end{bmatrix}
\begin{bmatrix}
\overrightarrow{AB}_y \\
\overrightarrow{AC}_y \\
\overrightarrow{PA}_y \\
\end{bmatrix} = 0
\end{cases}
$

矩阵方程可以解释为,寻找一条同时正交于(ABx, ACx, PAx)和(ABy, ACy, PAy)的直线(u,v,1).  
可以使用向量叉乘得到直线向量(x, y, z), 除以z值转换成(x/z, y/z, 1)的形式   

<font color=#00a8ff>重心坐标方程组：</font>
$\begin{cases}
  &w_1+w_2+w_3=1
  \\
  &P_x = w_1*A_x+w_2*B_x+w_3*C_x
  \\
  &P_y = w_1*A_y+w_2*B_y+w_3*C_y
  \\
\end{cases}$  


$w_2\overrightarrow{AB} + w_3\overrightarrow{AC} + \overrightarrow{PA} = 0$  

求$\vec{u} = cross(\begin{bmatrix}\overrightarrow{AB}_x&\overrightarrow{AC}_x &\overrightarrow{PA}_x\end{bmatrix},
\begin{bmatrix}\overrightarrow{AB}_y&\overrightarrow{AC}_y &\overrightarrow{PA}_y\end{bmatrix}
)$  

最后可得: 
$
\begin{cases}
w1 = 1 - (\vec{u}.x - \vec{u}.y)/\vec{u}.z
 \\
 w2 = \vec{u}.x/\vec{u}.z
 \\
w3 = \vec{u}.y/\vec{u}.z
\end{cases}
$  

<font color=#4cd137>当我们求出叉乘后的向量 $\vec{u}$ 后, 因为我们需要(u,v,1), 所以需要对u中的坐标除以u.z, 然后以相同的顺序对应到重心坐标方程组的w2和w3上, 最后根据w2和w3求出w1的值.</font>  

<font color=#4cd137>需要注意的是:在求叉积向量的时候,AB AC PA的顺序不同,w1 w2 w3对应的值也不同.</font>

<font color=#00a8ff>重心坐标代码：</font>

```cpp 
baryCentric(vec* pts, vec2 P)
{
	vec3 u = cross(vec3(pts[2][0] - pts[0][0], pts[1][0] - pts[0][0], pts[0][0] - P[0]), vec3(pts[2][1] - pts[0][1], pts[1][1] - pts[0][1], pts[0][1] - P[1]));
	if (std::abs(u.z) < 1) return vec3(-1, 1, 1);
	// 向量u的坐标需要满足(u, v, 1), 因此需要除以u.z转换成(u,v,1)
	// 同时重心坐标(w1, w2, w3)就可以对应, 注意这里, u.y和u.x的顺序
	return vec3(1.f - (u.x + u.y) / u.z, u.y / u.z, u.x / u.z);
}
```  

更简洁的重心坐标代码:  
```c
vec3 barycentric(vec2* pts, vec2 p)
{
    // AB AC PA
    vec2 pA = pts[0];
    vec2 pB = pts[1];
    vec2 pC = pts[2];

    vec3 v1 = vec3{ pB[0] - pA[0], pC[0] - pA[0], pA[0] - p[0] };
    vec3 v2 = vec3{ pB[1] - pA[1], pC[1] - pA[1], pA[1] - p[1] };
    // (u, v, 1) 对应 (w2, w3, 1-w2-w3)
    vec3 u = vec3(cross(v1, v2));
    u = u / u.z;

    // 重心坐标(w1 = 1-w2-w3, w2, w3)
    return vec3{ 1.f - (u.x + u.y), u.x, u.y };
}

```  