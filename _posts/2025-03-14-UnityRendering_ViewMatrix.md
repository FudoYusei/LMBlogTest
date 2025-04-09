---
layout: post
title: UnityRendering (六) View Matrix
subtitle: 世界空间转换到 View Space
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_ViewSpace.png
tags: UnityRendering
priority: 6
---

![banner](/assets/images/UnityRendering/UnityRendering_ViewSpace.png)  

View Space 也被称为 Camera Space, 但还是 View Space 更加准确. 当我们使用 Unity Editor 在 Scene View 中设置一个 Camera 对象, Scene 中的物体会从世界坐标系转换到摄像机的 Camera Space  

<span style='color:#4cd137'>而我们在 Scene View 中即能观察到 Camera 也能看到场景中的物体. 也就是说场景中的所有物体实际上也被转换到 Scene View Space 中. (也可以将这个视角看成一个额外的摄像机)</span>    

---

## 数学原理
Object Space 转换到 World Space 时, 计算对象空间基向量在 World Space 中的映射坐标  
World Space 转换到 View Space 时, 计算 WorldSpace 基向量在 View Space 中的映射坐标  

$$ M_{World}P_{World} = M_{View}P_{View} $$  
已知, View 在 WorldSpace 的坐标以及顶点在世界坐标系的坐标, 求顶点在 ViewSpace 的坐标.  
$$ P_{View} = M_{View}^{-1}M_{World}P_{World} $$

$ M_{World} $ 由基向量 $ [\vec{i} \vec{j} \vec{k}] $ 组成, $ M_{View} $ 是 View Space 在世界坐标系中空间矩阵的矩阵. <span style='color:#4cd137'>与 Object To World 的矩阵的求解过程刚好是反过来的.</span>  

就像为 3D 物体构造对象矩阵一样, 为 Camear 构造 View 矩阵, 然后使用它的逆矩阵将世界坐标系中的坐标转换到 View Space 中.  

---  

## View Space 的实现
Unity3D 的 View Space 不好说是怪异还是正统, 但是对于 View Space 的处理, 很好的体现了为什么说 Unity3D 渲染流水线使用了 OpenGL 风格.  
世界坐标系是左手坐标系, <span style='color:#e84118'>但 View Space 不是左手坐标系.</span> Unity3D 的 View Space 使用了和 OpenGL 一样的右手坐标系, <span style='color:#4cd137'>但是 NDC 坐标系却又是左手坐标系,</span> 屏幕空间算是右手坐标系.  

<span style='color:#4cd137'>因为 Unity3D 的世界坐标系是左手坐标系, 所以使用 Camera 的 Rotation Translation 构造出来的也是左手坐标系, 需要在 Scale 中翻转 z 轴, 变成右手坐标系.</span>  

<span style='color:#fbc531'>如果世界坐标系和 View Space 的手性相反, 那么叉乘规则改变, 是不是就没办法在 View Space 中进行叉乘计算?</span>  
<span style='color:#fbc531'>这样坐标系手性变化, 涉及到叉乘都需要搞清除相对偏手性, 不然计算出来的向量方向相反</span>  

代码实现, 大部分与 Object Matrix 的实现相同, 区别就是 z 轴取逆, 以及最后需要计算逆矩阵  
```csharp
    /// <summary>
    /// World to View Space
    /// </summary>
    /// <param name="camera"></param>
    /// <returns></returns>
    Matrix4x4 ViewMatrix(Transform camera)
    {
        // 1. Model matrix
        // 1.1 translate position
        Vector3 pos = camera.position;
        Vector4 trans = new Vector4(pos.x, pos.y, pos.z, 1);
        Matrix4x4 matTran = Matrix4x4.identity;
        matTran.SetColumn(3, trans);

        // 1.2. rotation
        Vector3 anglesInRad = camera.rotation.eulerAngles * Mathf.Deg2Rad;

        float cosX = Mathf.Cos(anglesInRad.x);
        float sinX = Mathf.Sin(anglesInRad.x);
        float cosY = Mathf.Cos(anglesInRad.y);
        float sinY = Mathf.Sin(anglesInRad.y);
        float cosZ = Mathf.Cos(anglesInRad.z);
        float sinZ = Mathf.Sin(anglesInRad.z);

        Matrix4x4 matZ = Matrix4x4.identity;
        Matrix4x4 matX = Matrix4x4.identity;
        Matrix4x4 matY = Matrix4x4.identity;

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

        // 1.3 view matrix Unity3D 使用的右手坐标系, OpenGL 风格
        Matrix4x4 matView = Matrix4x4.identity;
        // 右手坐标系反转z
        Matrix4x4 scaleZ = Matrix4x4.identity;
        scaleZ.m22 = -1;
        //matModel = matTran * matRotation;
        // matRotation.inverse = matRotation.tranpose
        // matTran.inverse != matTran.transpose
        // matTran 只需要 -trans

        // matCamera = matTran * matRotation * matScaleZ
        // 求逆矩阵
        matView = scaleZ.inverse * matRotation.inverse * matTran.inverse;

        return matView;
    }
```  
