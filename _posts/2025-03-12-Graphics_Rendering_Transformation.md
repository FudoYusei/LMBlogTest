---
layout: post
title: Computer Graphics (四) 手动实现软渲染-空间变换
subtitle: 3D 画面最终映射为 2D 画面
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
priority: 4
---

![banner](/assets/images/banners/ComputerGraphics/Projection.png)  

我们在屏幕上看到的"三维"画面实际上是假的, 只是看起来像是立体的三维画面, 是蒙骗大脑的视觉欺骗.  
图形渲染的起点是一个三维场景, 图形渲染的最终目的地是显示器上的二维屏幕. 所以图形渲染必不可少的部分就是空间变换.  

---  

## Spaces
空间转换的顺序如下:  
1. Object Space 对象空间, 3D 物体自身的坐标系, 图形渲染原始的输入数据, 就是对象空间的顶点坐标. 想象小孩子的玩具积木, 每个独立的积木就是一个 Object Space  
2. World Space 世界空间, 3D 场景的坐标系, 模拟真实三维场景. 想象成使用积木搭建一个场景  
3. View Space 观察者空间, 横看成岭侧成峰, 观察者相对于 3D 场景不同位置, 观察到的景象不同  
4. Clip Space 剪裁空间, 这是一个数学意义上的空间, 这里会经过透视转换, 也是三维空间真正转换为二维空间的步骤  
5. Screen Space 屏幕空间, 屏幕空间基本等效于屏幕的像素空间, 是最终内容呈现的坐标空间.  

<span style='color:#4cd137'>TinyRenderer 中并没有标准流程的 ViewSpce 和 ClipSpace</span>  


### 空间转换
每个空间的转换都存在一个转换关系, 数学上将转换关系归纳为矩阵计算. 关于空间转换矩阵的介绍很多, 这里只重点讲透视矩阵. <span style='color:#4cd137'>因为 TinyRenderer 作者使用的透视矩阵以及后续的成像原理与现在通用的理论完全不同</span>  

---  

## 透视
### 透视关系
TinyRenderer 作者使用的透视关系如下图所示:  
![Projection](/assets/images/banners/ComputerGraphics/Projection.png)  

TinyRenderer 中的透视矩阵并未基于 View Space, 而是直接基于 World Space. 并且将 World Space 的 x-z 平面当作投影平面.  

现代通用的透视关系, 使用锥形体模拟 3D 空间中有效视域: 近平面 远平面 vFov ratio  
![GeneralProjection](/assets/images/banners/ComputerGraphics/GeneralProjection.jpg)  

通常以近平面作为透视映射的平面(想象皮影戏观众看到的那块布), 此时深度值坐标是 [-n, -f], n 和 f 代表近远平面与原点之间的距离(标量, 没有正负号)  

TinyRenderer 的作者之所以可以不使用锥形体, 是因为压根就没有裁剪功能. 同时也启示我们透视映射的二维图像其实可以是任意一个平面  

> TinyRenderer 充分说明, 只要能将透视关系正确的整合到二维画面中, 随便自定义渲染流程都可以.  

### 透视矩阵
相机处于(0,0,c)处,将P点(x,y,z)投影至x-y平面上,得到投影点$P^`(x^`,y^`,z^`)$.  

<span style='color:#4cd137'>注意, 这与常见的透视投影到近平面上不同. TinyRenderer 直接在世界坐标系中进行透视投影</span>  

<span style='color:#4cd137'>Camera是C点. 映射平面在经过原点的xy平面上. 实际上, 映射平面在哪里都无所谓, 只要是按照比例计算出来的就行. 远近平面只是用来进行裁切的</span>  

根据相似三角形原理:  
$$ x^{'}=\frac{x}{1-z/c} $$  

$$ y^{'}=\frac{y}{1-z/c} $$  

同时,[x y z w]中,我们只要使得w = 1 - z/c  

构造矩阵  

$\begin{bmatrix} 
1 & 0 & 0 & 0
\\
0 & 1 & 0 & 0
\\
0 & 0 & 1 & 0
\\
0 & 1 & r & 1
\end{bmatrix}
\begin{bmatrix} 
x 
\\
y 
\\
z 
\\
1 
\end{bmatrix}=
\begin{bmatrix} 
x 
\\
y 
\\
z 
\\
rz+1 
\end{bmatrix}
$  

可得:  

$r = -\frac{1}{c}$  

---  

## 通用渲染流程
通用渲染流程的分步阶段, 包含了对应的渲染功能:  
1. Object To World Space: 包含了对象在 3D 空间中的位置转换. (Scale Rotation Translation)
2. World Space To View Space: 主要用于定义视域, 使用锥形体模拟现实世界中观察者有限的视野. 
3. View Space To Screen Space: 3D 空间映射为 2D 画面, 我们"看到"的现实世界, 实际上是大脑让我们看到的.

### Model  
Model将3D物体的坐标从对象空间转换到世界坐标系中.  

Model是Translation * Rotation * Scale  

存在一个通用的定理:  
$M_A M_B 都是相对于标准坐标系[\vec{i} \quad \vec{j} \quad \vec{k}]的矩阵$

$ M_A \vec{v}_a = M_B \vec{v}_b $  
$ \vec{v}_a = M_A^{-1} M_B \vec{v}_b $  

如果我们需要将Model坐标转换成世界坐标.  
通常世界坐标系本身就是标准坐标系. 因此逆矩阵和本身都是$ I $  
那么, 我们只需要知道的是模型在世界坐标系中所做出的SRT变化.  

---  

## View 
构造一个Camera矩阵, 将物体从世界坐标系转换到View坐标系, 也可成为相机空间   

同理, 要求出  
$$M_V \vec{v}_v = M_W \vec{v}_w$$  
$$ \vec{v} = M_V^{-1}M_W\vec{v} = M_V^{-1} \vec{v} $$  

---  

## Projection
透视矩阵, 将对象转换到一个齐次裁剪空间的立方体.  
之所以使用NDC是因为, 后续我们要将这些坐标转换到屏幕空间中, 而屏幕空间的分辨率是不一样的. 所以我们需要先标准化.  
在标准化之前, 需要先将世界坐标映射到近平面上.  
透视矩阵实际上就是一个将一个锥形体映射到一个齐次裁剪空间的立方体.  

原点就是我们的camera位置.  
实际上就是将view space中的点线性变换到NDC中.  
首先定义view space中的近平面: $z = -n$  

$ P_x 在view space中透视映射到近平面. $  
$ x = -\frac{n}{P_z}P_x $  
$ y = -\frac{n}{P_z}P_y $  

透视不会改变z值, 并不需要丢失原始值.  
$ z = P_z $  
此时已经进行了透视映射.  

然后我们需要标准化坐标到 <span style='color:#4cd137'>左右分别是l,r. 高低分别是t,b齐次线性空间. 映射到[-1,1]的坐标范围内</span>  
   

$ x 和 y $ 对应的是映射到近平面上的坐标. <span style='color:#4cd137'>注意, 现在还不是 Clip Space 中的坐标</span>  

下一步需要将x坐标从[r,l] 映射到[-1,1] ,将y坐标从[b,t]映射到[-1,1]. <span style='color:#4cd137'>同样也不是 Clip Space 中的坐标</span>  
  
所以:  
$ x^{'}=2 \frac{(x-l)}{r-l} - 1 $  
$ y^{'} = 2 \frac{y-b}{t-b} - 1 $  

z坐标从[-n,-f]映射到[-1,1], 因为我们在屏幕坐标线性插值的时候需要使用$\frac{1}{z}$ , 所以在顶点运算的时候直接进行$\frac{1}{z}$的插值  
线性插值的标准关系: $ z{'}=\frac{A}{z}+B $  
z取值[-n,-f], 对应的是[-1,1]  
$ -1 = \frac{A}{-n}+B $  
$ 1 = \frac{A}{-f} +B $  
求出:  
$ A=\frac{2nf}{f-n} $  
$ B=\frac{f+n}{f-n} $  
$ z{'} = - \frac{2nf}{f-n}(- \frac{1}{P_z} + \frac{f+n}{f-n}) $

代入x,y,z在view space中的表达式  
$ x^{'} = \frac{2n}{t-b}(- \frac{P_x}{P_z})- \frac{r+l}{r-l} $  
$ y^{'} = \frac{2n}{t-b}(- \frac{P_y}{P_z} - \frac{t+b}{t-b}) $  

构造一个点齐次坐标点$ P = <-x^{'}P_z, -y^{'}P_z, -z^{'}P_z, -P_z> $  

最终得到:  
$ -x^{'}P_z = \frac{2n}{r-l}P_x + \frac{r+l}{r-l}P_z $  
$ -y^{'}P_z = \frac{2n}{t-b}P_y + \frac{t+b}{t-b}P_z $  
$ -z^{'}P_z = \frac{f+n}{f-n}P_z - \frac{2nf}{f-n} $  

之所以是$-P_z$的原因在于, 这样矩阵中都是正数.  
$$ P^{'} = M_{frustum} P = 
\left[
    \begin{matrix}
        \frac{2n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
        0 & \frac{2n}{t-b} & \frac{t+b}{t-b} & 0 \\
        0 & 0 & - \frac{f+n}{f-n} & -\frac{2nf}{f-n} \\
        0 & 0 & -1 & 0
    \end{matrix}
\right]
\left[
    \begin{matrix}
    P_x \\
    P_y \\
    P_z \\
    1
    \end{matrix}
\right]
$$  

---  

## 齐次裁剪空间
为什么NDC又被称为齐次裁剪空间  
该阶段作为光栅化的最后一个阶段, 在光栅化钱裁剪, 减少顶点数量, 减少后续顶点运算量. 光栅化三个顶点组成一个三角形, 三角形会覆盖大量像素, 每个像素都是一个计算量.  

---  

## 透视除法  
https://stackoverflow.com/questions/24441631/how-exactly-does-opengl-do-perspectively-correct-linear-interpolation  

经过映射: $P^{'} = <-x^{'}P_z,-y^{'}P_z,-z^{'}P_z, -P_z>$  
透视除法就是除以$-P_z$或者w值  
由于透视插值的时候,   
$ \frac{1}{z_3}d = \frac{1}{z_1}(1-t)+\frac{1}{z_2}t $  
$ b_3 = z_3[\frac{b_1}{z_1}(1-t)+\frac{b_2}{z_2}t] $  

## 透视插值
右手坐标系中, 观察者或者说Camera所在的位置作为原点. x轴向右, y轴向上, z轴向外指向观察者. 也称为LookAt矩阵.  

对一个物体进行光栅化的步骤是:  
在近平面(z=-e)进行水平扫描, 每一条水平扫描线中分为等步长的点, 这个点代表着采样点.  
从观察者的位置(原点)发出射线通过水平扫描线上的采样点, 如果与物体(通常是三角形面片)相交, 计算物体在该采样点的属性(插值)  
根据透视还是正交视角, 决定是透视插值还是线性插值.  

三角形的顶点除了包含相机空间位置信息外, 还包含光照颜色和纹理映射坐标等信息.  
当绘制三角形的一条扫描线时, 每个像素的信息为左右两个端点的同类信息插值.  

顶点属性插值, 顶点属性插值本质上都是透视矫正, 因此都是和深度相关插值  

比如说光照, 本来照射到一个面上是同一个数据的关照, 但是有了深度就需要进行透视校正  
再比如说纹理坐标, 坐标也需要进行透视校正.  

## 光栅化   

三角形面上的一个点的深度坐标z可以由, 3D图形硬件进行线性插值得到. 权重坐标插值计算.  

光栅化一个三角形的之前, 先将三维空间中的三角形面片的三个顶点映射到近平面上(),物体已经经过空间变换得到一个NDC空间坐标. 然后将NDC坐标转换成屏幕坐标. 我们就获得了三角形在屏幕空间上的坐标. 也就是屏幕上的三角形.  
此时, 可以发现三角形的坐标从三维变成了二维, 也就是我们貌似丢失了深度信息, 而实际上, 深度信息依然保留了. 并没有被丢弃. 详见透视除法.  

现在, 我们拥有三角形的屏幕顶点坐标. 需要在屏幕上绘制三角形, 此时, 我们需要使用包围盒和扫描线找到三角形覆盖的屏幕像素, 也就是光栅化.  

计算扫描线上采样点相对于三角形的重心坐标.  
通过重心坐标判断该点处于三角形的内外.  
三角形内的采样点, 根据重心坐标插值得出该点属性. 而使用重心坐标插值这种线性插值, 会缺失透视矫正  

重心坐标的本质就是: $ \alpha \overrightarrow{AB} + \beta \overrightarrow{AC} = \overrightarrow{AP} $  
它也是一个线性插值. 光栅化之前, 我们能够得到的顶点属性以及额外的深度信息(插值必须使用). 这时候的顶点属性已经经过透视矩阵改变了.   
这时候会出现一个问题. 也就是所有的顶点都是经过透视插值. 而如果, 我们在光栅化的时候使用线性插值, 明显就有问题了.  
因此, 光栅化的时候依然要使用透视插值.  

然后, 逐行扫描屏幕三角形, 对于其中每一个像素点进行插值运算. 也就是所谓的光栅化.  

因此, 我们需要解决的一点就是, 在逐行扫描三角形时, 像素点的插值运算该怎么做.  

## 坐标大小关系
模型空间中的大小  
模型空间应该也是标准化[-1, 1]之间  

首先模型坐标系转换成世界坐标系  
使用标准矩阵, 也就是模型不动  
模型依然处于[-1 ,1]的模型空间中.  

## 什么时候透视除法
经过了MVP矩阵之后, 此时已经是NDC坐标.  
但是实际上为了计算方便.  
$ P^{'}=<-x^{'}P_z, -y^{'}P_z,-z^{'}P_z, -P_z$  

$ x^{'},y^{'},z^{'} $ 就是映射到[-1,1] NDC空间的坐标.  




## 二维和三维空间中的重心坐标
https://zhuanlan.zhihu.com/p/337296743  
三维重心坐标和二维不同的是, 三维重心坐标的P点必然要和三角形在一个平面上, 因此多了一个条件.  

二维重心坐标可以使用逆矩阵求解.  

三维空间中, 重心坐标插值本质就是线性插值, 因此同样需要进行透视矫正  

# 法向量的空间转换矩阵  

# viewport和screen space
https://blog.csdn.net/qq_28849871/article/details/73301766  

viewport 实际上就是NDC  

# 模型面显示不全
部分面显示不全, 一般三个因素影响  


显示依然有问题, 首先分清楚是uv坐标问题还是顶点问题  

## 查看uv坐标是否有问题
使用纯白色的颜色代替diffuseColor. 再次查看, 发觉是顶点坐标的问题, 有些面显示不对.  

## 查看深度值是否有问题 
取消zbuffer判断.  
取消后, 发觉还是一样的问题, 有些面显示不正确.  

## 判断顶点坐标的问题
顶点坐标现在是增加了三个矩阵乘法  

Model Space-> World Space 当前没有变化   
World Space -> View Space 构造一个简单的逆矩阵, 矩阵也没有错  
View Space -> NDC Space 融合了透视变化和归一化  

检查矩阵计算都没有问题.  

# 模型渲染大小
模型最终渲染的大小和摄像机形成的锥形体有着直接的关系.  

## viewspace的参数
摄像机常见的参数:  
position, 相机本身的位置. 使用世界坐标表示  
HFOV, Horizontal field of view, 代表摄像机水平方向的视野角度, 锥形体水平的夹角  
VFOV, Vertical field of view, 垂直方向, 默认  
near plane, 近平面距离(注意不是参数)  
far plane, 远平面  
ratio, 默认为高宽比, 由fov和近平面距离可以求出近平面的高, 然后借此求出宽, 影响物体最终渲染横向的大小.  

1. 首先, 近远平面不影响锥形体的透视关系和渲染大小, 只影响剪裁, 在物体能够完全渲染出来的情况下,不影响物体渲染的大小  
2. 物体的透视关系只和物体与摄像机的距离有关, 透视的射线角度不会变. 离得过近会有透视畸变发生.  
3. 物体最终渲染大小, 可以视为物体在锥形体中的比例大小. 当摄像机和物体的距离不变时, 物体映射到近平面上的大小不会变. 那么我们改变FOV, 近平面会跟随扩大和缩小, 而物体映射不会, 所以相对的物体最终渲染大小会改变.  
4. FOV以及Ratio本质上都是直接改变锥形体近平面的大小, 此时物体映射大小不变, 相对的物体在近平面上映射的大小发生改变. 视觉上, 物体在最终渲染效果上大小改变了.
4. 近平面不会影响锥形体的透视关系和渲染大小, 因为一旦近平面移动, 映射到近平面上的物体也会随着近平面等比例扩大缩小.  

这一套参数, nf代表剪裁的远近范围.  
摄像机位置代表远近关系(近大远小的透视), 过近会产生透视畸变.  
fov和ratio代表视野范围, 实际上就类似于放大缩小, 不会影响透视关系. 放大某个物体, 可以认为就是图像放大, 不会改变透视关系.  

## 另一套参数
对比另一套参数:  
position, 相机本身位置, 世界坐标表示  
near plane, 近平面距离  
height, 近平面的高度  
width, 近平面的宽度  

当相机与物体的距离固定的时候, 更改近平面距离, 会产生什么效果?  
由于近平面的宽高固定, 更改近平面距离就相当于视野范围放大缩小, <font color=#4cd137>只要不改变摄像机和物体的距离, 就不会影响透视关系</font>.  
然而近平面同时还影响远近范围剪裁.  
也就是说近平面同时影响了剪裁和视野范围.  

透视关系也会影响物体渲染大小.  

# clip vertices 和 screen coords  
首先, 获取模型面中的三个顶点  
三个顶点传入vertex shader中进行处理, 此时一个共用的 gl_Position 代表了顶点坐标从模型坐标系变换到齐次剪裁空间.   

光栅化时, 从齐次剪裁空间转换到viewport中, 此时已经是屏幕空间  

fragment shader得到屏幕坐标, 最终输出该像素点的颜色.  