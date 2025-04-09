---
layout: post
title: UnityRendering (五) 万向节死锁
subtitle: 万向节死锁到底影响了什么
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
excerpt_image: /assets/images/UnityRendering/UnityRendering_Gimbal.jpg
tags: UnityRendering
priority: 5
---

![banner](/assets/images/UnityRendering/UnityRendering_Gimbal.jpg)  

万向节死锁（Gimbal Lock）是使用欧拉角（Euler Angles）表示旋转时出现的一种现象，它会导致旋转自由度丢失，从而无法准确描述某些方向的旋转。这种现象在三维旋转中尤为常见，尤其是在机械系统（如万向节框架）或计算机图形学（如3D动画）中。
万向节死锁是旋转问题中非常常见的一个术语, 但是光看描述又难以理解.  

万向节死锁(Gimbal Lock) 到底丢失了什么旋转自由度? 为什么我们在 Unity Editor 中随意更改 transform 中的数据, 并没有遇到旋转角度的丢失?  

## 旋转
万向节死锁既然是物体旋转过程中发生的问题, 那么首先了解以下如何在 Unity3D 中控制物体的旋转.  

```csharp
namespace RotationDemo
{
    public class RotationDemo : MonoBehaviour
    {
        private Transform transform;
        private Vector3 originLocalRotation;
        public LocalRotation localRotation; 

        // Start is called before the first frame update
        void Start()
        {
            transform = GetComponent<Transform>();

            originLocalRotation = transform.localEulerAngles;
        }

        // Update is called once per frame
        void Update()
        {
            transform.localEulerAngles = originLocalRotation;

            // Local Rotation 顺序: Yaw -> Pitch -> Roll: y -> x -> z
            transform.Rotate(0, localRotation.Yaw, 0, Space.Self);
            transform.Rotate(localRotation.Pitch, 0, 0, Space.Self);
            transform.Rotate(0, 0, localRotation.Roll, Space.Self);
        }
    }

}
```  
物体旋转被解析成三个按照顺序的角度:   
1. Yaw, 绕着本体坐标 y 轴旋转.  
2. Pitch, 绕着本地坐标 x 轴旋转.  
3. Roll, 绕着本地坐标 z 轴旋转.  

<span style='color:#fbc531'>为什么旋转要被按照顺序解析成三个角度?</span>  
+ 从数学角度来说, 矩阵的乘法不具有交换律, 反应到现实就是, 旋转的先后顺序会得到不同的结果. 需要规定好先后顺序加上同样的角度才能得到同样的旋转结果.  
+ 三个角度, 因为本地坐标系等于逆序世界坐标系, 从世界坐标系的角度来看, 那必然需要这三个轴角才能得到所有角度的旋转结果.  

## 旋转顺序  
旋转顺序有一个让人容易混淆的地方: <span style='color:#4cd137'>只要旋转顺序不同, 就算旋转角度相同, 得到的结果大概率不同.</span> 因为矩阵乘法没有交换律.  
  
但是在 U3D Editor 的 Inspector 中更改物体 transform 组件中的 Rotation 属性, 只要有相同的角度, 必然得到相同的结果.   
<span style='color:#4cd137'>因为在 Editor 中, 修改 transform.Rotation 看似是动态分步修改, 实际上无论分多少次修改, 始终是三个固定顺序的矩阵相乘, 即官方文档提到的 zxy 顺序.</span> 和上一节中的代码实现方式类似, 分解成固定顺序的三个轴旋转.  

## 死锁的原因
死锁的原因是固定顺序的三个角度旋转. 矩阵乘法可以看作是对于空间基向量的线性变换.   
---
### **1.旋转矩阵**
内旋的矩阵组合顺序与旋转的应用顺序**相反**。对于 **Z→X→Y** 的内旋顺序：
- 先应用 **Z 轴旋转**，对应矩阵 \( R_z \)；
- 再应用 **X 轴旋转**，对应矩阵 \( R_x \)；
- 最后应用 **Y 轴旋转**，对应矩阵 \( R_y \)。

由于矩阵乘法是**从右到左**生效，总旋转矩阵为：
\[
R_{\text{total}} = R_y \cdot R_x \cdot R_z
\]

---

### **2. 具体矩阵形式**
- **绕 Z 轴旋转（γ）**（左手坐标系，顺时针为正）：
\[
R_z(\gamma) = \begin{bmatrix}
\cos\gamma & -\sin\gamma & 0 \\
\sin\gamma & \cos\gamma & 0 \\
0 & 0 & 1
\end{bmatrix}
\]

- **绕 X 轴旋转（α）**：
\[
R_x(\alpha) = \begin{bmatrix}
1 & 0 & 0 \\
0 & \cos\alpha & -\sin\alpha \\
0 & \sin\alpha & \cos\alpha
\end{bmatrix}
\]

- **绕 Y 轴旋转（β）**：
\[
R_y(\beta) = \begin{bmatrix}
\cos\beta & 0 & \sin\beta \\
0 & 1 & 0 \\
-\sin\beta & 0 & \cos\beta
\end{bmatrix}
\]

最终组合矩阵：
\[
R_{\text{total}} = R_y(\beta) \cdot R_x(\alpha) \cdot R_z(\gamma)
\]  

<span style='color:#4cd137'>矩阵的解释多种多样, 空间矩阵的列向量可以视为空间的基向量</span>  

$$ \begin{bmatrix}
    col_1 \cdot\vec{i} & col_2\cdot\vec{j} & col_3\cdot\vec{k}
\end{bmatrix}   $$  

旋转矩阵有一个共同点, 矩阵所形成的空间三个空间基向量里, 对应的旋转轴不变, 另外两个轴旋转.  




---

### 3. 全局坐标系  
全局坐标系会存在死锁吗? 死锁实际上与全局或者局部坐标系没有关系   
<span style='color:#4cd137'>在 Unity3D 全局坐标系中触发死锁的步骤如下:</span>  
1. 在 SceneView 中设置 Rotation Mode 为 Global, 保证 SceneView 中显示全局坐标系
2. 修改 Cube transform Rotation.x 为 90
3. 现在修改 Rotation.y 或者 Rotation.z, 观察 Cube 的旋转轨迹

死锁产生了, 无论如何修改 Rotation.y 或者 Rotation.z, Cube 只会沿着全局坐标 Y 轴旋转.  
按理说, 全局坐标系是静止不动的, 不应该产生死锁?  
但是, 实际使用中又确确实实的产生了死锁. 从数学角度来解释, Rotation.x 为 90, 则对应矩阵:  
\[
R_x(90) = \begin{bmatrix}
1 & 0 & 0 \\
0 & 0 & -1 \\
0 & 1 & 0
\end{bmatrix}
\]  

那么, 可以得到以下空间矩阵  
$$ R_{total} = R_y R_x(90) R_z(\gamma) = R_y  \begin{bmatrix}
\cos \gamma & -\sin \gamma & 0 \\
0 & 0 & -1 \\
sin \gamma & \cos \gamma & 0
\end{bmatrix} $$  

新的矩阵代表的旋转:  
1. 先绕着全局坐标 -Y 轴旋转 $ \gamma $  
2. 然后绕着全局坐标 Y 轴旋转 
数学角度上的死锁也产生了, $ \gamma $ 无法表示 Z 轴的旋转了. <span style='color:#4cd137'>由于 Z 轴旋转矩阵 $ R_z $ 是第一个应用的矩阵, 它的旋转会被后续的旋转所改变. </span>  

如果<span style='color:#4cd137'>按照 Z->X->Y 的顺序分步修改对应的旋转角度, 可以观察到 Cube 是正常的绕着对应的世界坐标轴旋转</span>  
如果<span style='color:#4cd137'>先修改 Y 旋转角度, 就会法线无论怎么修改 XY 的旋转角度, 都无法观察到 Cube 绕着对应的世界坐标轴旋转</span>  
  

---  

### 4. 局部坐标系死锁
运行 RotationDemo, 按照规定的顺序 Yaw(y) -> Pitch(x) -> Roll(z) 顺序修改三个角度, 会发现 Cube 完全遵循我们的预想, 围绕本地坐标轴旋转.  

对应的数学原理: $ P_w = M_z^{'} M_x^{'} M_y^{'} P_o = M_y M_x M_z P_o $   

<span style='color:#4cd137'>欧拉角旋转有一个隐藏的前提, 三个欧拉角即是世界坐标轴的旋转角度, 也是对应的本地坐标轴的旋转角度.</span> 我们可以根据欧拉角直接求出全局坐标轴的旋转矩阵, 但是没法直接求出本地坐标轴的旋转矩阵. 因此才有了上一篇中的两者转换的数学原理  

与全局坐标系相同的是, 设置 Pitch(x) 90 度后, <span style='color:#e84118'>无论如何更改 Yaw(y) 的值, Cube 都无法在本地坐标 y 轴上旋转.</span>    
无论怎么更改 Yaw(y) 的值, 都会被后序的旋转给异化.  

实际上这里体现了程序逻辑与现实的不同, <span style='color:#fbc531'>拿一支笔做示范, 将笔仰起 90 度, 手中的笔依然能够自由的 yaw pitch 和 roll, 并没有缺失角度.</span>    
现实中旋转并不是规定死的三次按照顺序的轴旋转, 而是可以无限叠加多次旋转, 所以不存在死锁.  

<span style='color:#fbc531'>那么为什么不采用这种无限叠加旋转的方式, 而非要使用三次固定顺序的旋转呢?</span>  
Unity3D 实际上内部旋转使用的是四元数旋转, 欧拉角只是用来更直观的显示当前物体的姿态.  

## 插值
### 欧拉角插值
目前在网上查到关于欧拉角插值的做法, 就是简单的对三个欧拉角插值, 优点是计算快, 缺点是插值路径不自然, 存在死锁风险, 只适用于角度变化小的场景.   

### 轴角
轴角插值, 可以代表任何旋转, 不会产生死锁, 但是不利于插值.  

### 四元数插值
四元数旋转就是轴角旋转的另一种形式, 可以代表旋转, 不会产生死锁, 并且利于插值, 唯一的缺点是难以理解  