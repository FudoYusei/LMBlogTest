---
layout: post
title: Unity Technology (四) Async Await
subtitle: Aysnc Await 好用的语法糖
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
excerpt_image: /assets/images/banners/UnityCoroutine/asynchronous-breakfast.png
tags: Unity3D
priority: 4
---

![banner](/assets/images/banners/UnityCoroutine/asynchronous-breakfast.png)  

Async Await 是 C# 提供的用于简化异步编程的一对关键字, 这对关键字一定是成对出现的. 编译器会识别 Async/Await 方法, 并将 Async/Await 内部转换成状态机.  
Async 修饰的方法表示该方法是异步操作, 该方法会被内部 Scheduler 接管运行(迭代器的 MoveNext 方法).  

---  

## 使用方法
```csharp
        static void Main(string[] args)
        {
            DoSomeThingAsyncly();
            // DoSomeTaskAsync();
            Console.WriteLine($"Main Thread at  {Thread.CurrentThread.ManagedThreadId}");

            Console.ReadKey();
        }

        async static void DoSomeThingAsyncly()
        {
            // 在 await 之前依然处于主线程
            Thread.Sleep(1000);
            Console.WriteLine($"Async Method Start at {Thread.CurrentThread.ManagedThreadId}");
            // 真正的异步方法
            await Task.Delay(1000);
            Console.WriteLine($"Async Method End at  {Thread.CurrentThread.ManagedThreadId}.");
        }
```  

---  

## Awaiter  
可以看到 async/await 组合中, await 常和 Task 一起连用  
```csharp
// call
DoSomeTaskAsync();

        // 异步方法
       async static void DoSomeTaskAsync()
       {
           Console.WriteLine("task start asynchronously");
           // await Task.Run()...
           // 运行异步方法
           var tast = Task.Run(() =>
           {
               for(int i = 0; i < 10; i++)
               {
                   Console.WriteLine(i);
               }
               // 阻塞线程 1s
               Thread.Sleep(1000);
           });

           // 同步执行结束
           await tast;
           // 同步后接着执行
           Console.WriteLine("task end after synchronization.");
       }
```  

`Task.Run()` 开启异步方法后, 返回 awaitable Task 对象. awaitable 对象就是对于异步方法的同步操作的抽象.  
<span style='color:#4cd137'>async/await 机制构造隐约的类似于 Unity3D 中协程, 但是总觉得有哪里不太一样.</span>  

对比 Unity3D 中协程的用法:  
```csharp
// call
DoSomeTaskAsync();

// call
void DoSomeTaskAsync()
{
    // 异步方法
    StartCoroutine(DoSomeThingAsync());
}


IEnumerator DoSomeThingAsync()
{
    Debug.Log("task start asynchronously");

    for(int i = 0;i < 10; i++)
    {
        Debug.Log(i);
        yield return null;
    }

    // 协程不是真的多线程, 阻塞协程等于阻塞主线程
    // 使用嵌套协程等待时间
    yield return new WaitForSeconds(1);
    Debug.Log("task end after synchronization.");
}
```  

<span style='color:#4cd137'>两者有一个明显的区别, async/await 模式需要手动开启异步方法. 例如调用 `Task.Run` </span>  

---  

## 内部机制
<span style='color:#fbc531'>async/await 本质上是语法糖, 编译器内部会将 Async/Await 方法转换成状态机代码, 状态机内部会将自己设置给 Scheduler.  </span>  

```csharp
using System;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        // 使用 async/await
        await MyAsyncFunction();

        // 使用模拟的 async/await
        await MyAsyncFunctionSimulated();
    }

    // 正常的 async/await 方法
    static async Task MyAsyncFunction()
    {
        Console.WriteLine("Starting async function...");
        await Task.Delay(1000); // 模拟异步操作
        Console.WriteLine("Async function completed.");
    }

    // 模拟的 async/await
    static Task MyAsyncFunctionSimulated()
    {
        var stateMachine = new AsyncStateMachine();
        stateMachine.Builder = AsyncTaskMethodBuilder.Create();
        stateMachine.Builder.Start(ref stateMachine);
        return stateMachine.Builder.Task;
    }
}

// 状态机模拟 async/await
struct AsyncStateMachine : IAsyncStateMachine
{
    public AsyncTaskMethodBuilder Builder;
    private int _state;
    private TaskAwaiter _awaiter;

    public void MoveNext()
    {
        switch (_state)
        {
            case 0:
                Console.WriteLine("Starting simulated async function...");
                _state = 1;

                // 模拟异步操作
                _awaiter = Task.Delay(1000).GetAwaiter();
                if (!_awaiter.IsCompleted)
                {
                    Builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;
                }
                break;

            case 1:
                Console.WriteLine("Simulated async function completed.");
                _state = -1; // 结束状态
                Builder.SetResult();
                return;
        }

        _awaiter.GetResult(); // 获取异步操作结果
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine)
    {
        Builder.SetStateMachine(stateMachine);
    }
}
```
  
---  

## 相关资料
[微软官方解释的异步工作流程.](https://learn.microsoft.com/zh-cn/dotnet/csharp/asynchronous-programming/)  

---  

[协程与多线程]: https://blog.csdn.net/weixin_44575037/article/details/105513014    
[IntelSample]: https://www.intel.com/content/www/us/en/developer/articles/code-sample/sample-application-for-direct3d-12-flip-model-swap-chains.html  
