---
layout: post
title: Computer Graphics (七) 手动实现软渲染-阴影映射
subtitle: 阴影是光影效果的核心
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
priority: 7
---

![banner](/assets/images/banners/ComputerGraphics/Projection.png)  

## 硬阴影
https://blog.csdn.net/wodownload2/article/details/103888380  
光源没有体积.  

渲染阴影相比于正常的渲染, 分为 two pass, 多出一步depth shader
depthshader, 将光源位置作为摄像机位置, 渲染场景中各个物体的深度  

## tinyrenderer中的实现 
```  
const float depth = 2000.f;     // 定义在our_gl.h, 为什么是2000?
struct DepthShader : public IShader {
    mat<3,3> varying_tri;

    DepthShader():varying_tri(){}

    vec4 vertex(const int iface, const int nthvert, vec4& gl_Position)
    {
        vec4 vertex = embed<4>(model->vert(iface, nthvert), 1);
        gl_Position = Viewport * Projection * ModelView * gl_vertex;    // transform it to screen coordinates
        varying_tri.set_col(nthvert, proj<3>(gl_vertex/gl_vertex[3]));
    }  

    virtual bool fragment(vec3 bar, TGAColor &color)
    {
        vec3 p = varying_tri * bar;
        color = TGAColor(255,255,255) * (p.z/ depth);
        return false;
    }
}
```  
```
void Projection(float coeff)
{
    Projection = Matrix::identity();
    Projection[3][2] = coeff;
}
```  
这一版本的triangle中没有计算带有透视的重心坐标, 因此, 所有坐标插值前需要进行透视除法  


## 正常的渲染流程  
1. 读取Model文件, 反序列化出Model的数据结构. 向下一步输入模型的面数 model.nface()
2. 遍历模型的每一个面, 得到每个面的三个顶点数据, 向vertexshader中输入当前模型面(面对应多个顶点, 一般是三角形) vec4 vertex(const int iface, const int nthvert, vec4& gl_Position)  
3. 默认的公共数据结构 vec4 clip_vert[3] , 默认情况下, vertexshader将模型的顶点坐标从object space转换到clip space. 这个数据默认由vertex shader写入, fragment shader读取. 实际上在这两个shader之间的光栅化过程中也要使用,  gl_Position = Projection*View*Model*vertexposition  
4. 一个面的顶点在vertexshader处理完之后, 结果被写入vec4 clip_verts[3]中, 接下来进行光栅化  
5. 光栅化先将这个面在剪裁空间的顶点坐标映射到屏幕坐标系中(透视除法), 然后通过包围盒得到这个面在屏幕坐标上的采样点(像素点), 遍历采样点, 得到该采样点的屏幕重心坐标, 并且求出对应的透视重心坐标(透视重心坐标使用在透视除法之前的插值上, 对于透视除法之后的插值直接使用重心坐标即可), 由fragment进行像素点处理, 返回像素的gl_color  

model -> face -> object space vertex - vertex shader -> clip vertex - rasterization  -> screen space vertex -> fragment shader -> color   

## 重心透视坐标和重心坐标
设三角形三个点的深度值为: $ z_1 \quad z_2 \quad z_3 $  
三个点对应的屏幕坐标的z值为: $z_1^{'} \quad z_2^{'} \quad z_3^{'} $  
当前屏幕采样点的重心坐标为:(a, b, c)  

重心透视坐标是用屏幕坐标的插值系数来求, 未透视除法之前的插值, 因此需要透视校正. 使用原始深度来弥补丢失的透视信息  

透视校正的重心坐标公式:  
$ P = 
\left[
    \begin{matrix}
    A & B & C
    \end{matrix}
\right]
\left[
    \begin{matrix}
    \frac{\frac{a}{z_1}}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}} \\
    \frac{\frac{b}{z_1}}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}} \\
    \frac{\frac{c}{z_1}}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}}
    \end{matrix}
\right]
$  

三个三维空间中的点$(x_1, y_1, z_1, V_1)(x_2, y_2, z_2, V_2)(x_3, y_3, z_3, V_3)$, 三个点映射在屏幕坐标系中, 在将三个点组成的面进行光栅化的过程中, 其中一个采样点的屏幕重心坐标为(a,b,c). 现在我们需要插值得到该点在三维空间中的值V  
$ V = \frac{\frac{a}{z_1} V_1}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}} + 
\frac{\frac{b}{z_1} V_2}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}} + 
 \frac{\frac{c}{z_1} V_3}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}}$  

而且, 将V更改成z, 三个点的进行插值的z值分别为$z_1 \quad z_2 \quad z_3$ 立马就能得到:  

$ \frac{1}{z} = a z_1^{'} + b z_2^{'} + c z_3^{'} $  
$ z = \frac{1}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}} $  
$ z = \frac{a+b+c}{\frac{a}{z_1} + \frac{b}{z_2} + \frac{c}{z_3}} $  

## shadowmap的渲染流程
shadowmap采用two-pass render  
first-pass 渲染阴影.  


### first pass
需要使用光源位置作为camera 生成一个lightViewSpace, 对object进行渲染, 并且相对于正常渲染来说, 阴影渲染只需要渲染深度值  

1. 根据光源位置构造一个ModelToView矩阵, 并且保存下来, second pass需要使用  
2. vertex shader就是正常功能, 将顶点从object space转换到clip space中  
3. 因为在fragment shader中, 已经是经过透视除法, 并且经过视口转换的屏幕坐标, 所以我们还需要在vertex shader中额外保存一下clip space的顶点坐标. varying_tri  
4. fragment正常的返回一个TGAColor 类型的值, 但是我们在这里返回并不是采样点的值, 而是采样点的深度. 通过透视校正插值得到clip_space的坐标, 然后进行透视除法获取NDC坐标中的深度值($z^{'}$), 这个深度值根据程序人为归一到一个范围内, 我们是归一到[-1,1].深度越大越远, 越小越近. 所以shadow map 越近越黑.  
```
mat<4,3> varying_tri;

virtual bool vertex(const int iface, const int nthvert, vec4& gl_Position)
{
    gl_Position = Projection * Projection * ModelView * embed<4>(model.vert(iface, nthvert), 1);
    varying_tri.set_col(nthvert, gl_Position);

    return false;
}

virtual bool fragment(const vec3 bar, TGAColor& gl_Color)
{
    vec4 clipPosition = varying_tri * bar;
    // 透视除法, 将深度值归一化到(-1,1), 但我们需要[0,1]
    gl_Color = TGAColor(255,255,255) * ((clipPosition[2]/clipPosition[3] + 1)/2);

    return false;
}
```  
<font color=#e84118>易错点:</font>  
需要明确深度值的范围. 我们需要将NDC坐标中的深度值从[-1,1]映射到[0,1]  
因为需要NDC坐标, NDC坐标与前面所有坐标系最大的不同就是透视除法  
只有没经过透视除法的坐标才能使用重心透视坐标.  
<font color=#4cd137>所以, vertex shader 需要将未经过透视除法的坐标传递给fragment Shader中, 实际上不需要clip space坐标, view 或者 Model都可以, 只要是透视除法之前的能进行透视插值</font>   
之后需要在fragment中进行透视除法和归一到[0,1]范围内.  

这样, 深度图就完成了, 深度图整体呈现白灰色, 离得越远越白, 越近越黑  

重心透视线性插值, 只要是列向量都可以插值.  
### second pass
1. 得到shadowbuffer之后, 进行second-pass, 大部分都是常见渲染流程, 多了一个全局矩阵, 用于将采样点的坐标从当前clip space转换到shadow clip space中, 获得shadow clip pos  
2. 我们需要得到屏幕坐标, 这样才能获取到shadow map中的值, 因此 shadow clip pos需要经过透视除法, 然后再进行viewport转换.
2. vertex shader没有变化, 依然是传递一个varying_clipPosition用于在fragmentshader中进行插值  
3. fragmentshader中, 多了一个shadow变量. 获取采样点在lightViewSpace中的深度值, 并且与shadowMap进行比较, 如果深度值大于shadowMap中的深度值, 说明该采样点无法被光源照射到, 因此需要加上阴影  


```
mat<4,4> uniform_MShadow;   // 从framebuffer的screenSpace转换到shadowbuffer的screenSpace

virtual bool vertex()
{
    // 同上
}

virtual bool fragment(const vec3 bar, TGAColor &color)
{
    vec4 screenPoint = varying_tri * bar;  // 插值获取当前采样点的screenSpace坐标
    vec4 shadowScreenPos = uniform_MShadow * screenPoint; // 获取当前采样点在shadow screen space中的映射点
    vec4 shadowDepth = shadowClipPos.z / shadowClipPos.w;   // 透视除法获取NDC坐标
}
```  
为什么要转换到屏幕空间, 因为我们需要在fragment中获取shadowMap中对应点的值.  
而shadowMap的坐标实际上就是屏幕坐标, 也就是NDC坐标进行视口转换后的坐标  
因为shadowMap就是要映射到屏幕上的图片, 所以shadowMap的坐标就是屏幕坐标  
从shadowMap中获取当前采样点映射的深度值.  
当前采样点的深度值如果大于shadowMap中映射点的深度值, 说明, 当前采样点位于光源照射不到的地方, 也就是处于阴影处.  

viewport矩阵一般来说, 不会改变深度值

## uniform前缀的含义  
https://zhuanlan.zhihu.com/p/33093968  

## varying前缀
https://zhuanlan.zhihu.com/p/103687720  


## tinyrender-shadow 中的坐标系 
### World Space
World 等于 Object Space  

### ViewSpace  
view space依然是右手矩阵  
z轴是center指向eye, y轴是up, x轴是right  

$ P^{'} = MP + T $  
数学上显而易见的逆操作是:  
$ P = M^{-1}P^{'}-M^{-1}T $  

然而从几何学上来说却反直觉.  
在经过M的线性变换之后(比如几何上来说, 3D空间中的旋转), 再进行位移T  
那么逆操作应该是经过$M^{-1}$的线性变换, 再进行位移-T, 即 $ P = M^{-1}P^{'}-T $  

那么问题出在哪呢?  
M这个操作是针对整个空间的, 比如旋转操作, 旋转的原点或者轴是世界坐标系的原点或者轴, 而不是本地坐标系.  
而我们惯性思维, 对移动了的点再次旋转, 实际上在点的本地坐标系中再次进行旋转.  

想要在本地坐标系进行旋转, 实际上是需要先逆转到世界坐标系. 因为我们的旋转操作都是针对于世界坐标系的.  
$ M_{world} M_{local} M_{world}^{-1} P $  
针对哪个坐标系的操作就需要先将世界坐标系中的点先转移到那个坐标系中.  

<font color=#e84118>那么为什么lookat矩阵使用</font> $ M^{-1}P - T $, 而不是$ M^{-1}P-M{-1}T $  
因为我们一直使用的是T=center=[0,0,0], 所以实际上两个式子相等.  

这个viewspace还有个非常反直觉的地方. <font color=#4cd137>一般来说, 我们构造camera的矩阵, 从eye处发出射线, 映射在近平面上.</font>  
<font color=#4cd137>而在这里, 我们构造center的矩阵, center指向eye, 从eye点发出射线, 映射在xoz平面上</font>  
离谱的是, 到底怎么控制映射坐标在[-1,1]之间的?  
<font color=#e84118>这种映射方式太混乱</font>  



### Projection
将xyz通过透视映射到[-1,1]上.  

$\begin{bmatrix}
1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & coeff & 1
\end{bmatrix}
\begin{bmatrix}
P_x \\ P_y \\ P_z \\1
\end{bmatrix} = 
\begin{bmatrix}
P_x \\ P_y \\ P_z \\1 + coeff*P_z
\end{bmatrix}$  

调用时, $coeff = \frac{-1}{(eye-center).norm()}$  

Projection 是怎么把xyz透视映射到[-1,1]上的.  



### Viewport
$ \begin{bmatrix} 
x + w/2 & 0 & 0 & w/2 \\ 
0 & y+h/2 & 0  & h/2 \\
0 & 0 & depth/2 & depth/2 \\
0 & 0 & 0 & 1
\end{bmatrix} 
\begin{bmatrix}
P_x \\ P_y \\ P_z \\1
\end{bmatrix}=
\begin{bmatrix}
x + w/2(P_x + 1) \\
y + h/2(P_y + 1) \\
depth/2(P_z + 1) \\
1
\end{bmatrix}
$  
可以看出在视口转换前, 坐标已经进入NDC中, 范围为[-1,1]  