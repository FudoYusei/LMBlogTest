---
layout: post
title: Unity3D Coroutine (五) Unity3D 协程执行机制
subtitle: Unity3D 多种类的协程执行机制
author: LM
categories: Unity3D
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
tags: Unity3D 
priority: 5
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine2.jpg)  

Unity3D 协程由迭代器模式实现, 协程调度器分步执行, 官方并没有详细的文档, 只能靠实例来了解相关机制  

## 协程调度的时机
最常使用的 yield return null, 调度时机在 Update 之后 LateUpdate 之前.  

```csharp
public class WhenCoroutine : MonoBehaviour
{
    private int updateCount = 0;
    private int lateUpdateCount = 0;
    // Start is called before the first frame update
    void Start()
    {
        StartCoroutine(RunInStart());
    }

    bool updated = false;
    // Update is called once per frame
    void Update()
    {
        if (!updated)
        {
            StartCoroutine(RunInUpdate());
            updated = true;
        }
        updateCount++;
    }

    bool lateUpdated = false;
    private void LateUpdate()
    {
        if (!lateUpdated)
        {
            StartCoroutine(RunInLateUpdate());
            lateUpdated = true;
        }
        lateUpdateCount++;
    }

    IEnumerator RunInStart()
    {
        Debug.Log($"Run In Start: Start at updateCount {updateCount}, lateUpdateCount {lateUpdateCount}");
        yield return null;
        Debug.Log($"Run In Start: End at updateCount {updateCount}, lateUpdateCount {lateUpdateCount}");

    }

    IEnumerator RunInUpdate()
    {
        Debug.Log($"Run In Update: Start at updateCount {updateCount}, lateUpdateCount {lateUpdateCount}");
        yield return null;
        Debug.Log($"Run In Update: End at updateCount {updateCount}, lateUpdateCount {lateUpdateCount}");
    }

    IEnumerator RunInLateUpdate()
    {
        Debug.Log($"Run In LateUpdate: Start at updateCount {updateCount}, lateUpdateCount {lateUpdateCount}");
        yield return null;
        Debug.Log($"Run In LateUpdate: End at updateCount {updateCount}, lateUpdateCount {lateUpdateCount}");
    }
}
```  

返回结果:  
Run In Start: Start at updateCount 0, lateUpdateCount 0  
Run In Update: Start at updateCount 0, lateUpdateCount 0

Run In LateUpdate: Start at updateCount 1, lateUpdateCount 0

Run In Start: End at updateCount 2, lateUpdateCount 1
Run In Update: End at updateCount 2, lateUpdateCount 1
Run In LateUpdate: End at updateCount 2, lateUpdateCount 1

<span style='color:#4cd137'>第一个特性: 普通的协程调度处于 Update 之后, LateUpdate之前</span>  
Run in Start 经过两个 Update 和 一个 LateUpdate, 说明 yield return null 这种普通的协程调度确实实在 Update 之后, LateUpdate 之前  

<span style='color:#4cd137'>第二个特性: 第一次开启协程, 会让协程立即运行一步, 不需要等待下一次调度</span>  
个人猜测, 因为 Unity3D 中迭代器的特性, 必然至少有一个 yield return 操作. 也就是至少可以调用 MoveNext 一次, 所以不需要等待下一次调度. 并且类比多线程, 开启新的线程也相当于立即执行异步操作  

## 嵌套协程同步  
嵌套协程同步后, 也是立即推进下一步, 不需要等待下一次调度, 符合现实的逻辑.   

```csharp
public class NestTest : MonoBehaviour
{
    bool lateUpdated = false;
    private void LateUpdate()
    {
        if(!lateUpdated)
        {
            StartCoroutine(Nested());

            StartCoroutine(NestedIEnumerator());

            lateUpdated = true;
        }
    }


    IEnumerator Nested()
    {
        Debug.Log($"Nested Start at Frame {Time.frameCount}");
        yield return StartCoroutine(Test());
        Debug.Log($"Nested End at Frame {Time.frameCount}");
    }

    IEnumerator Test()
    {
        Debug.Log($"Test Start at Frame {Time.frameCount}");
        yield return null;
        Debug.Log($"Test End at Frame {Time.frameCount}");
    }

    IEnumerator NestedIEnumerator()
    {
        Debug.Log($"NestedIEnumerator Start at Frame {Time.frameCount}");
        yield return TestIEnumerator();
        Debug.Log($"NestedIEnumerator End at Frame {Time.frameCount}");
    }

    IEnumerator TestIEnumerator()
    {
        Debug.Log($"TestIEnumerator Start at Frame {Time.frameCount}");
        yield return null;
        Debug.Log($"TestIEnumerator End at Frame {Time.frameCount}");
    }
}
```  
返回结果:  
Nested Start at Frame 1  
Test Start at Frame 1  

NestedIEnumerator Start at Frame 1  
TestIEnumerator Start at Frame 1  

Test End at Frame 2  
Nested End at Frame 2  

TestIEnumerator End at Frame 2  
NestedIEnumerator End at Frame 2  

LateUpdate 中执行开启协程的操作, 防止 Update 之后调度器周期执行混淆结果.   
<span style='color:#4cd137'>可以看到, 所有开启协程的操作都会立即执行第一个 MoveNext, 不需要等到下一个周期调度, 连嵌套协程也是</span>  


## 协程同帧返回  
```c
void OnEnable()
{
    StartCoroutine(_Do());
}

IEnumerator _Do()
{
    Debug.Log("[A]Frame " + Time.frameCount);
    yield return null;
    Debug.Log("[B]Frame " + Time.frameCount );
}
```  

返回结果:  
[A] Frame 0  
[B] Frame 1  

yield后面的语句需要等待yield执行完毕才能执行  

也就是MoveNext()首先执行到yield return null->设置Current为null->返回true  
然后下一帧继续调用MoveNext()->执行到函数末尾也就是yield后面的语句->没有设置Current的语句->返回false->等于协程执行结束,协程出栈  

因为MoveNext需要下一帧调用, 所以yield 之后的语句也只有下一帧执行.  

如果想要yield之后的语句本帧执行怎么办?  
包装一个IEnumerator, 内含 Stack<IEnumerator>, 就相当于一个小的协程执行机制  
只是它的MoveNext会检查Current的值, (假如我们特定yield return true为协程结束标识, 并且后面的语句同帧执行)  

```c
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using System;

public class CoroutineTest : MonoBehaviour
{
    void OnEnable()
    {
        StartCoroutine(ToFixedCoroutine(_Do()));
    }

    IEnumerator _Do()
    {
        Debug.Log("[A]Frame " + Time.frameCount);
        yield return true;
        Debug.Log("[B]Frame " + Time.frameCount);
    }

    public static IEnumerator ToFixedCoroutine(IEnumerator enumerator)
    {
        var parentsStack = new Stack<IEnumerator>();
        var currentEnumerator = enumerator;

        parentsStack.Push(currentEnumerator);

        while (parentsStack.Count > 0)
        {
            currentEnumerator = parentsStack.Pop();

            while (currentEnumerator.MoveNext())
            {
                var subEnumerator = currentEnumerator.Current as IEnumerator;
                if (subEnumerator != null)
                {
                    parentsStack.Push(currentEnumerator);
                    currentEnumerator = subEnumerator;
                }
                else
                {
                    // yield return true; 会设置Current为true. 我们让MoveNext再次执行一次
                    if (currentEnumerator.Current is bool && (bool)currentEnumerator.Current) continue;
                    yield return currentEnumerator.Current;
                }
            }
        }
    }
}
```  