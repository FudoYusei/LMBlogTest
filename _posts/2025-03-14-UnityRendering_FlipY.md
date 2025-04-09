---
layout: post
title: UnityRendering (八) FlipY
subtitle: 透视矩阵为什么需要 FlipY
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_TextureOrigin.jpg
tags: UnityRendering
priority: 8
---

![banner](/assets/images/UnityRendering/UnityRendering_TextureOrigin.jpg)  

Projection 透视是将 3D 空间转换为 2D 画面的关键过程. 简单来说, 根据相似三角形原理得到不同距离物体在映射平面上的缩放比例. 所谓的距离远近感实际上是大脑对于画面的解析信息, 也就是大脑让你认为的距离感. 这也是为什么就算照片是一个 2D 画面, 我们依然能够从照片中看出距离远近, 大脑"欺骗"了我们.  

---

## 数学原理
### 成像原理
标准渲染流程为 Camera 构建了一个视锥体, 用于表现人类视域的范围. 处于视域范围内的物体会被映射到近平面, 形成一个 2D 的映射画面, 也就是最终渲染到显示器上的画面.  

之所以能看到某个物体, 是因为接受到这个物体发射出来的光线  
Camera 所在位置就是我们接收光线的位置, 因为光线沿着直线传播, Camera 与物体的连线就是光线的传播路径, 映射平面(近平面)可以视为视网膜, 光线在近平面上成像. 可以发现透视关系就是简单的相似三角形原理. 缩放比例等于近平面距离/物体与 Camera距离.  

> 实际上人眼成像原理更接近小孔成像原理, 视网膜在瞳孔之后, 接受的图像是倒置的. 大脑自动给我们进行了翻转操作.  

### NDC
渲染流程中的透视投影矩阵合并了透视映射与映射到 NDC 的过程. 因为近平面作为透视映射的画面需要渲染到不同尺寸的显示器上, 近平面的尺寸是可以随时变化的, 显示器的尺寸也是可以随时变化的.  
编程理论告诉我们要将不断变化的部分抽象出来, 于是渲染到不同尺寸的显示器这一过程被抽象出来, 并且约定透视映射画面的标准, 也就是 NDC(统一设备坐标系).  

---  

## ProjectionMatrix 的实现
透视矩阵实际上将两步操作合二为一:  
+ 将 View Space 中的物体透视映射到近平面上
+ 将透视映射后画面再次映射到 NDC 中  

ProjectionMatrix 的计算过程的资料很多, 这里不再赘述. 标准的 OpenGL 风格透视矩阵公式:  
$$ P_{clipspace} = M_{clipspace}P = \begin{bmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\ 
0 & \frac{2n}{t-b} & \frac{t+b}{t-b} & 0 \\
0 & 0 & -\frac{f+n}{f-n} & \frac{2nf}{f-n} \\
0 & 0 & -1 & 0
\end{bmatrix} 
\begin{bmatrix}
    P_x \\
    P_y \\
    P_z \\
    1
\end{bmatrix}
$$  

<span style='color:#4cd137'>不同风格中 NDC 的定义也不相同, 主要区别是 z 值的范围.</span>  
OpenGL 的 z 值范围为 [-1, 1], DirectX 为 [0, 1]  
而 Unity Shader 检测到当前使用 DirectX API, 会开启 ReverseZ 功能, 为了使用 ReverseZ 功能, z 值必需映射到 [0, 1]  

---  

## Unity3D ProjectionMatrix
<span style='color:#e84118'>然而在 Unity3D 中使用上面的 ProjectionMatrix 并不能得到正确的渲染结果.</span>  
使用图形调试工具(Frame Debugger 或者 RenderDoc)会发现 Unity Shader 中的 UNIX_MATRIX_P 矩阵与我们的 ProjectionMatrix 不一样.  

> Unity Shader 本身也是编程语言, 所以文件结构和 CSharp 差不多.  
> 例如常用的函数 宏以及 global constant 都集中保存在一个文件中: UnityCG.cginc

区别有两点:  
1. y 轴坐标被反转, <span style='color:#4cd137'>这一点非常重要, 导致顶点顺序改变</span>  
2. 深度映射, 当 Unity3D 检测到使用 DirectX API, 就会开启 Reverse-Z 功能

> Reverse-Z 功能并不是一个配置选项, 而是一个功能, 需要使用者自己实现, 主要更改的点是 Projection 过程中反转近远平面, 设置深度的比较模式.  

### ReverseZ
简单说一下 ReverseZ, 浮点数的精度问题是因为小数点浮动导致不同大小的数字精度不同. 越接近 0 的位置精度越高, 远离 0 的位置精度越低.  

举例来说, 假设一个 8 bit 的浮点数. 去掉符号位 1 位, 指数位 2 位, 剩下 5 位有效数字. <span style='color:#4cd137'>关键点在于小数的有效数字位数是从小数点后第一个非零数字开始计算</span>  
0.000011111 有效位数 4 (默认有效位数前都为零)  
0.1111      有效位数 4  
0.100000001 有效位数 8 (无法省略中间的零)  

因为有效数字的机制, 越靠近 0, 有效数字前能够省略的零越多, 有效位数能够代表的数字越多, 也就是精度越高.  

在透视映射中, 默认近平面被映射到 0, 远平面被映射到 1. 原本远平面精度就低, 被映射到1, 同样精度不高. 如果远平面的物体比较靠近, 可能会出现深度精度不够, 看起来远平面会闪烁.  

ReverseZ 就是将精度低的远平面映射到精度较高的 0, 缓和精度问题, 这样就不会让精度雪上加霜.   

### 左右手坐标系
ReverseZ 会导致左右手坐标系改变吗? 确实会, 但是 剪裁空间中并没有涉及到叉乘操作, 只需要对 Z 值范围进行判断.  

## Flip Y
Unity3D 在使用 DirectX API 时, 做了特殊处理, 其中原理官方并未说明. 以下只是个人推导, 不一定正确.  
<span style='color:#4cd137'>如果使用的是 DirectX API, 那么屏幕坐标系原点默认在左上角, 并且无法更改.</span>  

<span style='color:#4cd137'>我们通常忽略了 Viewport 也会生成一个矩阵</span>  
  

---  

## 屏幕坐标系与 viewport
屏幕坐标系是 Graphic API 用于显示区域的坐标系. 不同风格的 API 使用不同的屏幕坐标系, 使用这些 API 只需要关注渲染内容映射到屏幕坐标系的屏幕坐标.  

### viewport
OpenGL 或者 DirectX 都会使用 API 设置 viewport 矩形区域, <span style='color:#4cd137'>并且无法更改原点位置, 原点位置是不同风格 API 的重要区别.</span>  

Viewport 矩形区域只需要设置左右两个对角点.  
```csharp
SetViewport(leftX, leftY, rightX, rightY);
```  

<span style='color:#4cd137'>标准渲染流程中, 与屏幕坐标系相关的就是 Viewport, Viewport 就是屏幕坐标系中的一块矩形区域.</span>  

### 屏幕坐标系
Unity 采用的是 OpenGL 风格, 默认屏幕空间原点处于左下角, 而 DirectX 默认屏幕空间位于左上角.     

最终渲染内容停留的坐标系就是屏幕坐标系.  

---  

## 透视除法
### what
在渲染流程计算中, 顶点坐标从一开始就是四元格式 (x, y, z, w). 之所以称为透视除法, 就是顶点坐标全部除以 w. 因为在透视映射之后, w = -Pz, 除以 w 得到透视映射的坐标.  

### why
<span style='color:#fbc531'>为什么会存在透视除法? 直接将 ViewSpace 映射为 NDC 坐标不行吗?</span>  

个人猜测, 这么做的目的是源自于数学计算考虑. 从 View Space 映射到 NDC 坐标的公式如下:  
$$ x^{'} = - \frac{2n}{r-l}(-\frac{P_x}{P_Z}) + \frac{r+l}{r-l} $$  
$$ y^{'} = - \frac{2n}{r-l}(-\frac{P_y}{P_Z}) + \frac{t+b}{t-b} $$  
$$ z^{'} = - \frac{2nf}{f-n}(-\frac{1}{P_Z}) + \frac{f+n}{f-n} $$  

但是这样的映射关系, 无法使用矩阵表达, 将 -Pz 提取出来之后, 就可以得到透视矩阵, 并且这个矩阵还可以直接与四元格式的顶点坐标相乘.  

$$ P_{clipspace} = M_{clipspace}P = \begin{bmatrix}
\frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\ 
0 & \frac{2n}{t-b} & \frac{t+b}{t-b} & 0 \\
0 & 0 & -\frac{f+n}{f-n} & \frac{2nf}{f-n} \\
0 & 0 & -1 & 0
\end{bmatrix} 
\begin{bmatrix}
    P_x \\
    P_y \\
    P_z \\
    1
\end{bmatrix}
$$  

> 矩阵与四元格式的顶点坐标相乘, 从一开始对象空间转换到世界空间就已经在使用, 在这个流程中使用也是统一计算格式, 最大化硬件并行计算效率.  

### when
有一种说法, 因为四元顶点格式 w = -Pz, 透视除法除以 w, 所以透视映射会丢掉原始深度坐标信息. 所以当我们还需要使用原始深度信息时, 还无法进行透视除法.  

使用透视矩阵将顶点坐标从 View Space 映射到 Clip Space, 会根据管线设置剪裁掉在设置范围外的顶点, 并且生成新的切点, 原始深度值 w 用于进行裁剪范围判断. <span style='color:#4cd137'>裁剪空间中, $ x_{clip} y_{clip} z_{clip} 的合理范围都在 [-w, w] 之间. 可以使用一个四元格式坐标存储裁剪所需要所有信息 $ </span>  

剪裁操作完毕后, 进入光栅化阶段, 将 NDC 中的三角形转换成屏幕三角形, 对于屏幕三角形覆盖的像素 ,光栅化时需要透视矫正插值, 透视矫正插值公式:  
$$ b_3 = z_3[(\frac{b_1}{z_1})(1-t) + \frac{b_2}{z_2}t] $$

z 代表原始深度值, b 代表想要插值的属性值. 任何会受到原始深度影响的都需要在光栅化时进行透视矫正插值.  

<span style='color:#e84118'>这里存在矛盾的地方, 透视除法可以得到 NDC 坐标, 再通过 NDC 坐标转换为屏幕坐标. 再通过屏幕三角形光栅化, 透视矫正插值得到插值的像素点信息</span>  
那么问题就是, 透视除法真的丢掉了原始深度信息 w 值吗? 只需要验证 Fragment Shader 输入信息的 w 值即可.  

### FragmentShader  
```C
Shader "Unlit/ShowXYZW"
{
    Properties
    {
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };


            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // 原始深度值范围就是视锥体的近远平面 (0.3, 1000)
                // float c = i.vertex.w/ 1000;

                // x y 此时都处于屏幕坐标系
                float c = i.vertex.x/ 800;
                
                return fixed4(c, c, c, 1.0f);
            }
            ENDCG
        }
    }
}

```  
给场景中的 3D 物体赋予材质之后, 就可以移动 3D 物体的位置来观察颜色变化. 使用取色工具可以得到颜色的具体数值, 范围在 [0-255].  

<span style='color:#4cd137'>颜色分量值在 (0,1) 之间, 所以xyzw需要除以对应的范围值, 不然大于 1 时颜色就是白色</span>  

通过颜色反馈可以得到 FragmentShader 的输入数据:  
+ x y 都是屏幕坐标系坐标, 代表像素在屏幕坐标系的坐标, 取值范围是屏幕坐标系的范围
+ w 是原始深度值, 代表像素可见的原始深度, 取值范围是 Camera 视锥体的近远平面

<span style='color:#4cd137'>也就是说透视除法之后并没有丢失原始的深度信息, 依然存储在 w 值中</span>  

> 由于 View Space 右手坐标系的构造, 物体深度值实际上都是负值, 而公式中 w = -Pz, 所以这里原始深度值实际上并不准确, 但是更容易理解 

## 附录: UnityCG
Unity3D Shader 常用的变量都放在 UnityCG.cginc 中, 想要使用其中的变量或者函数, 使用 include 语句包含该文件即可.   
熟悉而常用的各种 Unity_MATRIX 就定义在这里. <span style='color:#4cd137'>通常为了多平台通用, 变量和函数会做很多处理.</span>  