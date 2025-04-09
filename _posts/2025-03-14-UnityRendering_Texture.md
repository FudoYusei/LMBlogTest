---
layout: post
title: UnityRendering (十) 翻转的 Texture
subtitle: 为什么 Unity3D 需要翻转 uv.y
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
priority: 10
---

![banner](/assets/images/UnityRendering/UnityRendering_TextureOrigin.jpg)  

Opengl 与 directx 一个明显的区别就是屏幕空间. Opengl 以左下角为原点, DirectX 以左上角为原点. 这个定义最大的影响就是 Texture, 因为 Texture 的存储受到原点影响.  

---

## 原点的影响
[D3D 与 OpenGL 中 Texture 原点的影响](https://www.puredevsoftware.com/blog/2018/03/17/texture-coordinates-d3d-vs-opengl/)  
[Texture Origin](https://alek-tron.com/TextureOrigins/TextureOrigins.html)  

<span style='color:#4cd137'>使用 OpenGL 或者 DX 的图形 API 上传图片到 Texture 资源中. Texture 可以视为显存中的二维数组, 通过 uv 坐标可以获取其元素 </span>  

DX 采用的是左上角原点, 存储布局友好. OpenGL 采用左下角原点, 数学计算布局友好.  


### 渲染 3D 世界
<span style='color:#4cd137'>纹理坐标只会影响纹理在显存中的布局, 两者的显存布局 y 轴上下颠倒.</span> 在 shader 中常见的计算关系式: 1 - uv.y  

如同上面的两个链接所示, 原点的差异并没有影响 3D 世界的渲染结果. <span style='color:#4cd137'>因为 3D 世界中的纹理采样的锚定点是 uv 坐标, 这句话非常重要.</span>  

### 显示图片纹理
当我们想要使用在渲染流程中使用贴图时, 就已经确定了纹理坐标系. 例如, 一个正对着摄像机的 Quad, 想要让它显示一张图片纹理.  

#### 情况一
为了简化过程, 图片纹理 (0, 0) 处为黑色, (1, 1) 处为白色  
1. 构造一个 Quad 用来显示图片纹理
   1. 在 OpenGL 中, 左下角为原点. 那么 Quad 左下角 uv 坐标为 (0, 0)  
   2. 在 DirectX 中, 左上角为原点, 那么 Quad 左上角 uv 坐标为 (0, 0)  
   3. OpenGL 与 DirectX 中 uv 坐标 y 轴翻转 1 - uv.y 
2. 加载图片到内存中, 并调用图形 API 上传至显存中的 Texture 资源
   1. 在 OpenGL 中, 图片的第一个数据元素 (0, 0) 被加载至 Texture 的 (0, 0), 左下角  
   2. 在 DirectX 中, 图片的第一个数据元素 (0, 0) 被加载至 Texture 的 (0, 0), 左上角  
3. 根据 uv 坐标对图片纹理采样
   1. 在 OpenGL 中, 左下角采样得到黑色
   2. 在 DirectX 中, 左上角采样得到黑色

因为 Quad 相当于使用 uv 坐标模拟了双方的屏幕坐标系, 渲染结果在 y 轴上翻转, 符合双方屏幕坐标系. 图片是否上下颠倒就取决于这张图片本身的原点位置, 因为大部分图片格式都是左上角为原点, 所以基本上是 OpenGL 进行 uv 处理.  

<span style='color:#4cd137'>双方各自使用符合自身定义纹理坐标的 3D 模型与纹理, 渲染过程与结果符合预期</span>  

#### 情况二
使用相同的 3D 模型  
1. 构造同一个 Quad 用来显示图片纹理, 以 OpenGL 为基准
   1. 在 OpenGL 中, 左下角为原点, Quad 左下角 uv 坐标为 (0, 0)
   2. 在 DirectX 中, 使用同一个 Quad, 左下角 uv 坐标为 (0, 0)
2. 加载图片到内存中, 并调用图形 API 上传至显存中的 Texture 资源
   1. 在 OpenGL 中, 图片的第一个数据元素被加载至 Texture 的 (0, 0), 左下角  
   2. 在 DirectX 中, 图片的第一个数据元素被加载至 Texture 的 (0, 0), 左上角  
3. 根据 uv 坐标对图片纹理采样
   1. 在 OpenGL 中, 左下角 uv 坐标 (0, 0), 采样图片纹理 (0, 0) 为黑色
   2. 在 DirectX 中, 左下角 uv 坐标 (0, 0), 采样图片纹理 (0, 0) 为黑色

<span style='color:#4cd137'>使用相同的 3D 模型, 相同的纹理坐标, 居然得出了相同的渲染结果.</span>   

DirectX 使用 OpenGL 坐标的模型,  相当于 uv 在 Y 轴翻转, 也就是情况一渲染结果 Y 轴翻转.  

#### 情况三
在情况二的基础上, 将渲染结果拷贝到目标纹理, 创建一个新的 Quad, 同样使用 OpenGL 风格, .  
1. 情况二中相同的 3D 模型与纹理坐标, 上传相同的图片作为纹理, 得到相同的结果  
2. 现在要将渲染结果拷贝到指定的 Texture 中, 构造一个覆盖整个屏幕的 Quad, 附加目标纹理, 渲染结果现在作为输入纹理, 纹理存储布局遵循 Opengl 和 DirectX 各自的纹理坐标系布局
   1. 在 OpenGL 中, 输入纹理(0,0) 在左下角, 采样依然是黑色
   2. 在 DirectX 中, 输入纹理(0,0) 在左上角, 采样不是黑色, 拷贝到新的目标纹理中, 图案再次反转

<span style='color:#4cd137'>情况三, 就是 Unity3D 使用 DirectX API 的情况</span>  
+ 3D 模型的纹理坐标是 OpenGL 风格
+ 但是因为使用的是 DirectX API, 所以, 纹理存储布局使用的是 DX 风格.
这就导致了渲染正常, 但是渲染到 Texture 就会反转.  

## Unity3D Texture
<span style='color:#4cd137'>Unity3D 官方说明不需要使用者处理 OpenGL 与 DX 平台的差异.</span>  

### Texture
Unity3D 加载图像文件会自动导入成 Texture 资产, 使用 RenderDoc 查看渲染过程, 可以看到导入的 Texture 在 DX API 下是上下颠倒, 而在 OpenGL API 下是正常的. 个人猜测, Unity3D 在导入过程中自动做了处理, 保证在调用 OpenGL API 的时候得到正常的纹理(因为大多数图像文件符合 DX 风格, 所以 Opengl 的输入纹理一般是颠倒的)  

---  

## Unity3D 的特殊透视矩阵
Unity3D 标准渲染流程有一个特殊的地方. 使用 RenderDoc 可以看到 Camera.Render 中分为好几个步骤:  
1. Camera.RenderOpaque, 渲染不透明物体
2. Camera.RenderTransparency, 渲染半透明物体
3. Camera.ImageEffects, 屏幕后处理

而第三步, 屏幕后处理仅仅调用 Hidden/BlitCopy 将前两个步骤渲染的 Texture 拷贝到一个新的目标 Texture 中.  

Unity3D 在渲染过程中采取的是 OpenGL 风格. 只要是 Unity Engine 生成的渲染内容, 屏幕坐标系原点都是左下角.  

当使用 OpenGL API 时, 一切正常, 符合标准渲染流程.  
但是使用 DirectX API 时, 从透视矩阵开始就显得与标准流程不同, 但是实际上就是情况二加上情况三.  

Unity3D 遵循 OpenGL 风格, 但是使用 DX API, 如同情况二, 渲染内容没有差别, 可以正常显示在屏幕上.  

但是, 还有一个屏幕后处理, 在使用 DX API 时, 会将 DX 风格的纹理上下翻转.   

<span style='color:#4cd137'>于是, 在情况二的渲染过程中, Unity 的特殊透视矩阵对 y 轴映射取负值, 让渲染结果上下颠倒</span>    
在情况三中, 拷贝到目标 Texture 中, 再次颠倒, 负负得正回到正常渲染结果.  

---  