---
layout: post
title: UnityRendering (四) Model Matrix
subtitle: 对象空间转换到世界空间
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_Overview.png
tags: UnityRendering
priority: 4
---

![banner](/assets/images/UnityRendering/UnityRendering_Overview.png)  


## 数学原理
矩阵变换坐标的本质是将顶点从从一个基坐标空间转换到另一个基坐标空间中, 顶点的位置不变, 只是坐标改变.  

旋转和缩放都是三维空间中的线性变换的一种表现形式, 矩阵就是[线性变换](https://blog.csdn.net/Xuuuuuuuuuuu/article/details/120488971)的一种表现形式. 之后加上平移操作, 被称为仿射变换.  

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

因此, 想要得到对一个空间进行线性变换. 只要得到源空间的基向量变换到目的空间的基向量, 就可以组成对应的转换矩阵  
<font color=#e84118>基向量改变, 求线性变换的矩阵</font>  
$$ \vec{v}_{world}  = M_{objectToworld} \vec{v}_{object} $$  

同时, 也可以从上面等式得到, <span style='color:#4cd137'>矩阵即可以看作是对点坐标的线性变换, 也可以看作是对空间基向量的线性变换.</span>  
例如 $ M_{rotationx} P $ 既可以看作是将点 P 绕着世界坐标轴 X 旋转, 也可以看作是将本地坐标轴绕着世界坐标轴 X 旋转得到一个新的本地坐标轴.  

已知对象空间的坐标, 使用线性变换, 将坐标变换到世界坐标系中指定的位置.  
我们要求出$ M_{world} $, 只要得到对象的基向量在世界坐标转换后的向量即可.  
这里, 对象空间基向量为 $ \vec{i} \vec{j} \vec{k} $  

<font color=#e84118>注意区别: 世界坐标系中的同一个顶点转换成view space坐标系表示, 也就是顶点没有变换, 换了基向量.</font>  
针对世界空间转换到View Space中:  
$$ M_{view} \vec{v}_{view} = M_{world} \vec{v}_{world}  $$  
$$ \vec{v}_{view} = M_{view}^{-1} M_{world} \vec{v}_{world}  $$  
只要二者的变换矩阵都是基于同一个基向量即可直接相乘变换.  

## Object Space
对象空间, 3D 对象的局部坐标系, 通常以模型中心或者特定锚点(脚底)为原点.  

## World Space
如果仅限于如何使用 Unity3D Editor 开发游戏, 那么毫无疑问 World Space 是核心点.  
所有的可渲染对象都需要放置在 World Space 的某个地方, 而这个放置操作具体到数学关系中就是空间矩阵变换.  

对象空间转换到世界空间的操作被定义为, 需要严格按照顺序:  
1. 缩放, 沿着 <span style='color:#4cd137'>世界坐标系</span>的 x y z 轴缩放
2. 旋转, 绕 <span style='color:#4cd137'>本地坐标系</span>的 zxy 轴旋转
3. 位移, 在世界坐标系中位移

<span style='color:#fbc531'>为什么要严格按照顺序执行? </span> 
1. 缩放操作会改变基向量长度, 但不会改变基向量方向.  
2. 旋转操作会改变基向量方向, 不会改变原点位置.
3. 位移操作需要基于原点在世界坐标系中位移.

---  

## Rotation
当一开始旋转的时候, 本地坐标系和世界坐标系是重合的, 旋转相当于基于世界坐标系, 会改变本地坐标轴的方向.  
在 Unity3D 中调用 `Rotate` 顺序是: zxy 顺序. 但是, U3D 官方并没有说是按照本地坐标轴旋转还是世界坐标轴旋转. <span style='color:#4cd137'>默认是围绕世界坐标系的轴旋转, 但是可以通过参数变更世界坐标系或者本地坐标系</span>  
  

### 世界坐标轴和本地坐标轴  
[绕世界坐标系的轴旋转和本地坐标系的轴旋转是可以互相转换的](https://zhuanlan.zhihu.com/p/656702979)   

$ R_x R_y R_z $ 三者都是基于世界坐标系空间下绕xyz三个轴旋转. 也就是所谓的静态欧拉角, 轴是固定的  
动态欧拉角, 想象物体的本地坐标系. 会随着旋转一起旋转. 假设本地坐标系的三个轴为  $ \vec{i} \vec{j} \vec{k} $    
当经过了$ R_x $ 的旋转后, 本地轴也会跟随旋转变为 $ R_x\vec{i} , R_x\vec{j} ,R_x\vec{k} $  

绕世界坐标系y轴旋转的矩阵是 $ R_y $, <font color=#4cd137>这里假定了一个等式关系, 本地轴和世界轴的旋转角度相同</font>. 例如, 世界坐标系X轴旋转$ \theta $, 那么本地坐标系x轴也要旋转 $ \theta $ . 这样就可以在已知 $ R_x $ 的情况下来表示本地x轴的旋转.  
  
先定义本地坐标系中的旋转顺序x->y->z, 对应的旋转矩阵$ r_x r_y r_z $.  

<span style='color:#4cd137'>推导过程会重复并且穿插的使用矩阵对点或者对坐标轴的影响</span>  

x轴最先旋转, 此时x轴还是与世界坐标系重合, 因此  $ r_x = R_x $  
然后是y轴, 本地y轴受到本地x轴旋转影响, 已经不与世界坐标系重合了, <span style='color:#4cd137'>但是, 现在只有绕着世界坐标轴 Y 旋转的矩阵, 所以需要想办法用这个矩阵代替本地坐标轴 y 的旋转矩阵.</span>   

想要使用 $R_y$, 那么必需保证使用的时候, 本地坐标轴 y 与世界坐标轴 Y 重合. 
1. 点 P 绕着本地坐标 x 轴旋转, 矩阵 $ r_x $ 的意义就是, 可以让原先与世界坐标轴重合的本地坐标轴旋转为 $ M_{x}^{'} $, 所以先逆转本地 x 轴旋转, 让坐标系回到最初与世界坐标系重合的状态 
2. 应用 $ R_y $ 矩阵旋转点, 得到一个新的经过了世界坐标 Y 轴旋转的点.  
3. 然后再使用矩阵 $r_x$ 将与世界坐标轴重合的本地坐标系旋转为 $ M_{x}^{'} $, 绕着世界坐标 Y 轴旋转的点也被映射成绕着本地坐标系 $ M_{x}^{'} $ 的 y 轴旋转  

$ r_y = r_xR_yr_x^{-1} $  
同理, 对于本地坐标 z 轴的旋转:  
1. 需要先将经过了 $r_yr_x$ 旋转的本地坐标轴, 逆转($ (r_yr_x)^{-1} $)回与世界坐标轴重合. 
2. 然后应用世界坐标 Z 轴的旋转矩阵得到旋转点. 
3. 再使用 $ r_yr_x $ 将与世界坐标轴重合的本地轴再次旋转  

$ r_z = r_yr_x R_z r_x^{-1}r_y^{-1} $  

最后$ r_zr_yr_x = R_xR_yR_z $  
因此得到结论, 在本地坐标轴上的旋转, 可以化成相反顺序的全局坐标轴的旋转. 世界坐标轴的旋转顺序可以转换成相反的本地坐标轴的顺序.   


### 缩放
再进行缩放操作, 此时缩放操作虽然基于世界坐标系, 世界坐标系与本地坐标系重合, 因此等于在本地坐标轴上进行缩放.  

### 旋转 
旋转矩阵是针对世界坐标系的坐标轴进行旋转, 那么会出现一个问题, 更改 GameObject 的 Transform 的 Rotation 应该会出现不符合预期的结果.  

例如, 先将 Cube 沿着 x 轴旋转 30 度,   

在缩放操作之后, 本地坐标轴被拉长, 但是坐标轴方向未变, 对于旋转来说, 世界坐标系依然与本地坐标系重合, 所以旋转矩阵依然可以使用.  

### 位移
最后进行位移操作, 对象的位移都是基于世界坐标系. 位移操作并不是线性变换, 相当于是独立在坐标轴外的操作.   

### ModelMatrix
对象的 Scale Rotation 和 Translation 都会影响到对应的 ModelMatrix. 使用 ModelMatrix 的地方, 在 VertexShader 中将顶点数据转换至屏幕坐标系的过程中, 第一次空间转换就是 Object->World, 使用的就是 ModelMatrix  

---  

## 绘制一个三角形  
构造一个 Triangle Mesh, Mesh 是由三角形面组成的网格体  
+ 网格体使用到的所有顶点  
+ 三角形面使用的三个顶点索引
+ 顶点绕序, 顶点索引的顺序包括了顶点绕序
+ 顶点属性, 需要插值的顶点属性

### 顶点
三角形三个顶点 ABC, A(0, 0, 0) B(0, 1, 0) C(1, 0, 0)  

### 顶点索引
顶点索引 {0, 1, 2}, 当观察者位于 -Z 方向时, 三角形面的顶点绕序是顺时针, 也就是正面.  

### 顶点绕序-顺逆时针
三角形面顶点的顺逆时针排列决定了面的法向量朝向, 而我们规定法向量方向指向观察者, 就说明这个面是正面.  

Unity3D 的世界坐标系是左手坐标系, 那么顶点顺序是顺时针的三角形面视为正面, 不会在渲染时被剔除.  

<span style='color:#fbc531'>那么如果对象坐标系是右手坐标系, 右手坐标系的面转换到左手坐标系中到底会怎么样?</span>  

**第一种情况** 已知世界坐标系是左手坐标系, 但对象坐标系并没有声明自己的偏手性  
那么对象坐标系默认和世界坐标系同样的偏手性, 因为这时候对象坐标系就是纯数学运算, 数学运算在左右手坐标系中通用.  
例如, 对象坐标系中三个基向量 (1, 0, 0) (0, 1, 0) (0, 0, 1) 转换到左手坐标系中, 它就构成了左手坐标系, 转换到右手坐标系中, 它就构成了右手坐标系.  
<span style='color:#4cd137'>对象坐标系绝大多数情况下手性, 使用建模软件的时候, 就已经用 Front Up Right 三个方向锚定了 xyz 三个基向量, 除非是纯靠理论建模</span>  

**第二种情况** 已知世界坐标系是左手坐标系, 对象坐标系给出了自己的偏手性  
当建模软件将模型输出为特定的格式, 比如 Unity3D 使用的 fbx 格式文件, 就会有对应的选项 Front Up Right 对应的轴方向  

<span style='color:#4cd137'>左手坐标系的形式各不相同</span> 以 Unity3D 左手坐标系来说, Front 为 +Z, Up 为 +Y, Right 为 +X. 同时, Front 为 +X, Up 为 +Z, Right 为 +X 也是左手坐标系  

<span style='color:#4cd137'>也就是说, 与人类认知 3D 空间的方式相同, Front Up Right 三个方向就是观察者用于锚定的基准左手坐标系</span>  

<span style='color:#fbc531'>那么, 右手坐标系的面转换到左手坐标系, 偏手性会改变吗?</span>  
按照常理来说, 面不会改变, 就像摆放积木到场景中, 我们预期, 并不会改变积木本身的性质, 这个性质指的是基向量构成的空间.   
但是, 原先面会使用左手坐标系的坐标来表示, 顶点坐标不仅仅是三个数字, 还有隐藏的基向量. 只要转换成了左手坐标系中的坐标, 那么基向量跟着改变, 偏手性当然也会改变.  

<span style='color:#4cd137'>但是, 容易让人混淆的是数学计算, 尤其是叉乘. 叉乘会经常省略掉基向量.</span>  
   

**举例说明左右手坐标系转换的本质**  
假设对象空间: Front 为 -Z, Up 为 +Y, Right 为 +X 组成的右手坐标系, A(0, 0, 0) B(1, 0, 0) C(0, 1, 0) 三个顶点组成一个三角形面, 顶点是逆时针排列.  

<span style='color:#4cd137'>观察者的位置也会影响三角形面的正反, 观察者处于 +Z 方向与 -Z 方向观察一个面的顶点绕序是相反的, 默认观察者处于 +Z 方向</span>  

求三角形面的法向量: $ \vec{n} = \vec{AB} \times \vec{AC} = (0, 0 ,1) $, +Z 方向, 三角形的法向量朝向观察者, 三角形就是正面. 右手坐标系中面顶点逆时针排列代表面的法向量指向观察者  

假设世界空间: Front 为 +Z, Up 为+Y, Right 为 +X 组成的左手坐标系, ABC 三个点从对象坐标系转换到世界坐标系, 只需要将点的 Z 值取负值. 也就是 A'(0, 0, 0) B'(1, 0, 0) C'(0, 1, 0) 三点, 依然是逆时针排列.  
求三角形面的法向量: $ \vec{n} = \vec{A^{'}B^{'}} \times \vec{A^{'}C^{'}} = (0, 0 ,1) $, +Z 方向, 数学计算并不会改变, 但是左手坐标系的 +Z 方向改变, 因此导致三角形法向量方向也跟着改变, <span style='color:#e84118'>意味着现在面的朝向被改变了, 与我们本意不符</span>  

$ \vec{AB} 和 \vec{A^{'}B^{'}} $ 向量的数学坐标相同, 看起来像是同一个向量, 但实际上不同, 因为双方的基向量不同, 这也是为什么看起来相同的叉乘, 在左右手坐标系中得到不同的向量.   

上面叉乘看起来求出的是 (0, 0, 1), 实际上求出来的是基向量 $ \vec{k} $, 在左右手坐标系转换的时候, 选择给 $ \vec{k} $ 取反. (左右手坐标系表现形式很多, 转换方式也不同)  

<span style='color:#4cd137'>左右手坐标系转换, 不仅要转换坐标轴的值, 还需要逆转顶点绕序</span>  

---  

### 实现代码
Unity3D 中创建一个三角形 Mesh, 并使用默认的 Material 进行渲染.  

```csharp
public class GenerateTriangle : MonoBehaviour
{
    public Material material;
    // Start is called before the first frame update
    void Start()
    {

        gameObject.AddComponent<MeshFilter>();
        MeshRenderer meshRenderer = gameObject.AddComponent<MeshRenderer>();
        Mesh mesh = GetComponent<MeshFilter>().mesh;

        mesh.Clear();

        // 以当前gameobject.position为中心点, 构造一个三角形

        // 世界坐标系为左手坐标系 所以顺时针为正面
        // 为了简化情况, 对象坐标系也设置为左手坐标系, 这样不需要转换
        mesh.vertices = new Vector3[] {new Vector3(0,0,0), new Vector3(0,1,0), new Vector3(1,0,0)};
        mesh.uv = new Vector2[] { new Vector2(0, 0), new Vector2(0, 1), new Vector2(1, 1) };

        // 顺时针顺序
        mesh.triangles = new int[] { 0, 1, 2 };

        //var material = Resources.Load<Material>("Materials/TransformationMat");
        //var material = Resources.Load<Material>("Materials/UnlitMaterial");
        // 设置网格体渲染所需要的 Material
        meshRenderer.material = material;

    }
}
```  
---  

## 自定义 ModelMatrix
标准的渲染流程中, Vertex Shader 将顶点坐标从对象空间转换到 Clip Space 中, 接下来我们会逐步使用自定义的矩阵代替这个过程.  

ModelMatrix 实际上就是从对象坐标系转换世界坐标系的映射, 按照顺序进行 Scale Rotation 和 Translation  

在 Unity3D 中, 可以通过 GameObject 的 Transform 获取 Scale Rotation 和 Translation 的数据.  
没有什么需要说明的地方, 就是将数据赋值到对应的矩阵上, 唯一的问题就是找错极其麻烦.  

```csharp
   /// <summary>
   /// Model to World Space
   /// </summary>
   /// <param name="goTransform"></param>
   /// <returns></returns>
   Matrix4x4 ModelToWorldMatrix(Transform goTransform)
   {
       // 1. Model matrix
       Matrix4x4 scaleMat = Matrix4x4.identity;
       Vector3 scale = goTransform.localScale;
       scaleMat.m00 = scale.x;
       scaleMat.m11 = scale.y;
       scaleMat.m22 = scale.z;

       // 1.1 translate position
       Vector3 pos = goTransform.position;
       Vector4 trans = new Vector4(pos.x, pos.y, pos.z, 1);
       Matrix4x4 matTran = Matrix4x4.identity;
       matTran.SetColumn(3, trans);


       // 1.2. rotation
       Vector3 eulerAngles = goTransform.rotation.eulerAngles;
       Vector3 anglesInDegree = eulerAngles * Mathf.Deg2Rad;

       float cosX = Mathf.Cos(anglesInDegree.x);
       float sinX = Mathf.Sin(anglesInDegree.x);
       float cosY = Mathf.Cos(anglesInDegree.y);
       float sinY = Mathf.Sin(anglesInDegree.y);
       float cosZ = Mathf.Cos(anglesInDegree.z);
       float sinZ = Mathf.Sin (anglesInDegree.z);

       Matrix4x4 matZ = Matrix4x4.identity;
       Matrix4x4 matX = Matrix4x4.identity;
       Matrix4x4 matY = Matrix4x4.identity;
       float v = matZ[0,1];
       
       // z 旋转
       matZ.m00 = cosZ;
       matZ.m01 = -sinZ;
       matZ.m10 = sinZ;
       matZ.m11 = cosZ;

       matX.m11 = cosX;
       matX.m12 = -sinX;
       matX.m21 = sinX;
       matX.m22 = cosX;

       matY.m00 = cosY;
       matY.m02 = sinY;
       matY.m20 = -sinY;
       matY.m22 = cosY;

       Matrix4x4 matRotation = matY * matX * matZ;

       // 1.3 Model matrix
       Matrix4x4 matModel= Matrix4x4.identity;

       // 按照 Scale Rotation 和 Translation 的顺序
       matModel = matTran * matRotation * scaleMat;

       return matModel;
   }
```  
---  

## 使用 ModelMatrix
现在已经得到 ModelMatrix, 接下来就是使用 ModelMatrix. 在 Shader 中自定义矩阵的 uniform 全局变量  

```c
Shader "MyShader/TransformationShader"
{
    Properties
    {
        _MyColor("Main Color", Color) = (1,1,1,1)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            HLSLPROGRAM
            #pragma enable_d3d11_debug_symbols
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            uniform float4x4 MatModelToWorld;
            uniform float4x4 MatWorldToView;
            uniform float4x4 MatViewToClip; 
            float4 _MyColor;

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
                o.vertex = mul(mul(MatViewToClip, mul(MatWorldToView, MatModelToWorld)), v.vertex);
                return o;
            }

            float4 frag (v2f i) : SV_Target
            {
                return _MyColor;
            }
            ENDHLSL
        }
    }
}

```  

在代码中给 uniform 变量赋值  

```csharp
    void Update()
    {
        Camera mainCamera;
        if(Camera.current != null)
        {
            mainCamera = Camera.current;

        }
        else
        {
            mainCamera = Camera.main;
        }
        Debug.Log(mainCamera.ToString());

        // uniforrm 变量赋值
        material.SetMatrix("MatModelToWorld", ModelToWorldMatrix(this.transform));
        material.SetMatrix("MatWorldToView", ViewMatrix(mainCamera.transform));

        GraphicsDeviceType deviceType = SystemInfo.graphicsDeviceType;
        if (deviceType == GraphicsDeviceType.Direct3D11)
        {
            material.SetMatrix("MatViewToClip", ProjectionMatrixForD3D(mainCamera.fieldOfView, mainCamera.aspect, mainCamera.nearClipPlane, mainCamera.farClipPlane));

        }
        else
        {
            material.SetMatrix("MatViewToClip", ProjectionMatrix(mainCamera.fieldOfView,    mainCamera.aspect, mainCamera.nearClipPlane, mainCamera.farClipPlane));
        }
    }
```  

---  