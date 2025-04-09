---
layout: post
title: Unity Coroutine (一) 异步与多线程
subtitle: 异步编程与多线程编程
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
tags: Unity3D 游引擎
priority: 1
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine1.jpg)  

异步编程是一种编程范式, 允许程序在等待某些操作完成时执行其他任务, 而不是停下来等待. 常见的网络请求与 I/O 操作, 如果游戏客户端在主逻辑中等待这些操作, 游戏界面并卡死.  

<span style='color:#4cd137'>想要将方法组合起来实现功能, 那么方法之间必然是存在耦合, 例如 方法 A 调用方法 B. 异步方法必然存在同步, 例如 方法 A 等待异步方法 B 完成. 两者本质是一样的.</span>  

## 异步编程
常见的多线程就是异步编程的一种, 现代编程语言中, 创建一个多线程并不难, 但是想要多线程能够运转起来却不简单.  

```csharp
        static void Main(string[] args)
        {
            int sum = 0;
            int i = GetNumI();
            int j = GetNumJ();
            Console.WriteLine(i + j);
        }

        private static int GetNumJ()
        {
            return 1;
        }

        private static int GetNumI()
        {
            return 2;
        }
```  

使用 Thread 实现带有返回值异步方法, 需要手动实现的功能:  
1. `RunAsync`, 单独开辟一个线程执行方法, 并且返回该线程的同步对象  
2. `AsyncResult` 类, 专门用于线程同步的对象, 内部使用事件机制进行同步  

在主线程中新开两个异步方法, 并且等待返回结果后, 将两者相加.  
```csharp
using System;
using System.Threading;

class Program
{
    static void Main(string[] args)
    {
        // 调用包装后的异步方法
        var asyncResult1 = RunAsync(() => CalculateSum(10, 20));
        var asyncResult2 = RunAsync(() => CalculateSum(30, 40));

        // 主线程等待两个异步方法完成并获取结果
        int result1 = asyncResult1.GetResult();
        int result2 = asyncResult2.GetResult();

        // 计算并输出结果
        int totalSum = result1 + result2;
        Console.WriteLine($"Result of Thread 1: {result1}");
        Console.WriteLine($"Result of Thread 2: {result2}");
        Console.WriteLine($"Total Sum: {totalSum}");
    }

    // 将线程包装成异步方法
    static AsyncResult<int> RunAsync(Func<int> func)
    {
        var asyncResult = new AsyncResult<int>();

        // 创建一个线程来执行任务
        Thread thread = new Thread(() =>
        {
            try
            {
                int result = func(); // 执行传入的函数
                asyncResult.SetResult(result); // 设置任务结果
            }
            catch (Exception ex)
            {
                asyncResult.SetException(ex); // 设置异常
            }
        });

        thread.Start(); // 启动线程
        return asyncResult;
    }

    // 模拟一个耗时的计算函数
    static int CalculateSum(int a, int b)
    {
        Thread.Sleep(1000); // 模拟耗时操作
        return a + b;
    }
}

// 自定义异步结果类
class AsyncResult<T>
{
    private T _result;
    private Exception _exception;
    private ManualResetEvent _waitHandle = new ManualResetEvent(false);

    // 设置结果
    public void SetResult(T result)
    {
        _result = result;
        _waitHandle.Set(); // 通知等待的线程
    }

    // 设置异常
    public void SetException(Exception exception)
    {
        _exception = exception;
        _waitHandle.Set(); // 通知等待的线程
    }

    // 获取结果（阻塞直到结果可用）
    public T GetResult()
    {
        _waitHandle.WaitOne(); // 等待结果
        if (_exception != null)
        {
            throw _exception; // 如果有异常，抛出
        }
        return _result; // 返回结果
    }
}
```  

<span style='color:#fbc531'>上面的代码实际情景中很少使用, 因为在主线程中等待同步, 等同于应用程序"卡死", 主线程需要继续运行. 所以不应该在主线程中等待事件, 而是直接判断状态</span>  

```csharp
using System;
using System.Threading;

class Program
{
    static void Main(string[] args)
    {
        // 调用包装后的异步方法
        var asyncResult1 = RunAsync(() => CalculateSum(10, 20));
        var asyncResult2 = RunAsync(() => CalculateSum(30, 40));

        // 主循环，检查异步任务是否完成
        while (true)
        {
            // 核心点: 检查第一个任务是否完成 & 检查第二个任务是否完成.
            // 主线程只检查共享的状态, 所以不会被堵塞, 这是非常常用的模式.
            if (asyncResult1.IsCompleted && syncResult2.IsCompleted)
            {
                Console.WriteLine("Task 1 completed! Task 2 completed!");

                // 主线程等待两个异步方法完成并获取结果, 同步机制
                int result1 = asyncResult1.GetResult();
                int result2 = asyncResult2.GetResult();
                break;
            }

            // 模拟主线程的其他工作
            Console.WriteLine("Main thread is doing other work...");
            Thread.Sleep(500); // 模拟主线程的其他操作
        }

        // 计算并输出结果
        int totalSum = result1 + result2;
        Console.WriteLine($"Result of Thread 1: {result1}");
        Console.WriteLine($"Result of Thread 2: {result2}");
        Console.WriteLine($"Total Sum: {totalSum}");
    }

    // 将线程包装成异步方法
    static AsyncResult<int> RunAsync(Func<int> func)
    {
        // 核心点: 通过闭包的形式让内存共享在多线程中
        var asyncResult = new AsyncResult<int>();

        // 创建一个线程来执行任务
        Thread thread = new Thread(() =>
        {
            try
            {
                int result = func(); // 执行传入的函数
                asyncResult.SetResult(result); // 设置任务结果
            }
            catch (Exception ex)
            {
                asyncResult.SetException(ex); // 设置异常
            }
        });

        thread.Start(); // 启动线程
        return asyncResult;
    }

    // 模拟一个耗时的计算函数
    static int CalculateSum(int a, int b)
    {
        Thread.Sleep(1000); // 模拟耗时操作
        return a + b;
    }
}

// 自定义异步结果类
class AsyncResult<T>
{
    // 异步结果
    private T _result;
    private Exception _exception;
    // 同步事件
    private ManualResetEvent _waitHandle = new ManualResetEvent(false);

    
    // 是否完成
    public bool IsCompleted
    {
        get
        {
            lock (_lock)
            {
                return _isCompleted;
            }
        }
    }

    // 设置结果
    public void SetResult(T result)
    {
        _result = result;
        _waitHandle.Set(); // 通知等待的线程
        _isCompleted = true;
    }

    // 设置异常
    public void SetException(Exception exception)
    {
        _exception = exception;
        _waitHandle.Set(); // 通知等待的线程
        _isCompleted = true;
    }

    // 获取结果（阻塞直到结果可用）
    public T GetResult()
    {
        _waitHandle.WaitOne(); // 等待结果
        if (_exception != null)
        {
            throw _exception; // 如果有异常，抛出
        }
        return _result; // 返回结果
    }
}
```  
可以看到, 手动实现异步方法的同步机制非常麻烦, 简单的功能都需要一大串代码, 实际上 C# 提供了对应的 Task 类.  

<span style='color:#4cd137'>上面代码是一种多线程常用的模式</span>  
1. 封装一个 AsyncResult 类, 存储异步操作状态和结果
2. 封装一个异步方法, 手动开启多线程  

## 相关资料
[微软官方解释的异步工作流程.](https://learn.microsoft.com/zh-cn/dotnet/csharp/asynchronous-programming/)  

---  

[协程与多线程]: https://blog.csdn.net/weixin_44575037/article/details/105513014    
[IntelSample]: https://www.intel.com/content/www/us/en/developer/articles/code-sample/sample-application-for-direct3d-12-flip-model-swap-chains.html  
