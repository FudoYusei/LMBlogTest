---
layout: post
title: Unity3D Coroutine (八) Unity3D Task
subtitle: Unity3D 中正确的使用 Task
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
priority: 8
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine2.jpg)  

多线程无法访问 Unity3D 主线程(UI 线程)的资源, 导致 Task 和 async/await 机制无法正常的作用于 Unity3D 中. 但是, 我们可以进行自定义扩展, 让 Unity3D 也可以正常的使用 Task 和 async/await 机制.  

## TaskScheduler  
TaskScheduler 顾名思义, 用于 Task 的调度, 觉得 Task 什么时候以及在哪里执行.  
一个典型的 Task 可以粗略的分为三个部分:  
+ 主线程, 或者说调用 Task.Run 的线程  
+ Task 主体方法
+ Task 后续回调方法


```csharp
Console.WriteLine($"Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");
Task.Run(() => {
    Console.WriteLine($"Task Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
}).ContinueWith((ac) =>
{
    Console.WriteLine($"Continue Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
}).ContinueWith((ac) =>
{
    Console.WriteLine($"Continue Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
});
```  
其中一个返回结果:  
Main Thread ID: 1  
Task Run on Thread ID: 6  
Continue Run on Thread ID: 6  
Continue Run on Thread ID: 6  

这些方法都是由 TaskScheduler 进行调度. Task 主体方法和回调方法都处于同一个线程.   

自定义一个 TaskScheduler 的扩展类来定制调度行为  

```csharp
        public class perThreadTaskScheduler : TaskScheduler
        {
            protected override IEnumerable<Task> GetScheduledTasks()
            {
                return null;
            }

            protected override void QueueTask(Task task)
            {
                var thread = new Thread(() => { TryExecuteTask(task); });

                thread.Start();
            }

            protected override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
            {
                throw new NotImplementedException();
            }
        }

        public void Main()
        {
            Console.WriteLine($"Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");

            var task = Task.Factory.StartNew(() =>
            {
                Console.WriteLine($"Task Start at {Thread.CurrentThread.ManagedThreadId}");
            }, CancellationToken.None, TaskCreationOptions.None, new perThreadTaskScheduler());

            task.ContinueWith(
                 (t) => { Console.WriteLine($"Continue at {Thread.CurrentThread.ManagedThreadId}"); }, CancellationToken.None, TaskContinuationOptions.None, new perThreadTaskScheduler());
        }

```  

其中一个返回结果:  
Main Thread ID : 1  
Task Start at 11  
Continue at 12  

因为自定义 PerThreadScheduler 代替了默认的 Task 调度  

## SynchronizationContext
SynchronizationContext 与 TaskScheduler 的区别在哪?  
TaskScheduler 用于异步调度, [SynchronizationContext 用于异步操作之间的消息同步](https://www.cnblogs.com/xiyin/p/15004355.html).  

<span style='color:#4cd137'>SynchronizationContext 等于运行一段操作所必须的上下文, 线程 A 将自己的 SynchronizationContext 共享出来, 线程 B 就可以向 A 的 SynchronizationContext 发送操作, 让这些操作在线程 A 上执行. </span>  

### 回调函数
回调函数究竟由谁执行? 通常我们说需要注册一个回调函数:  

```csharp
namespace SynchronizationContextDemo
{
    internal class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine($"Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");

            //CallbackThread ct = new CallbackThread(Callback);

            //// async / await
            //DoAsync(Callback);

            //// Task
            //Task.Run(() =>
            //{
            //    Console.WriteLine($"Task Start on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
            //    Thread.Sleep(1000);
            //}).ContinueWith((t) =>
            //{
            //    Callback();
            //});

            //Task.Run(() =>
            //{
            //    Console.WriteLine($"Task Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
            //    Thread.Sleep(1000);
            //}).ContinueWith((ac) =>
            //{
            //    Console.WriteLine($"Continue Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
            //    Thread.Sleep(1000);
            //}).ContinueWith((ac) =>
            //{
            //    Console.WriteLine($"Continue Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
            //    Thread.Sleep(1000);
            //});

            Task.Run(() => {
                throw new Exception("hahaha");
            }).ContinueWith((t) =>
            {
                Console.WriteLine(t.Exception);
            }, TaskContinuationOptions.run);

            Console.WriteLine("Main Thread End");
            Console.ReadKey();
        }

        static void Callback()
        {
            Console.WriteLine($"Callback Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        }

        async static void DoAsync(Action callback)
        {
            Console.WriteLine($"Do Async Start on Thread ID: {Thread.CurrentThread.ManagedThreadId}");

            await Task.Run(() => { 
                Console.WriteLine($"Do Async Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
                Thread.Sleep(1000);

            });

            // 同步后调用
            //Console.WriteLine($"After Do Async, callback run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
            callback();
        }
    }



    public class CallbackThread
    {
        public CallbackThread(Action callback)
        {
            Thread t = new Thread(() =>
            {
                Console.WriteLine($"Thread Run on Thread ID: {Thread.CurrentThread.ManagedThreadId}");
                Thread.Sleep( 1000 );
                // 执行 callback 
                callback();
            });

            t.Start();
        }
    }
}
```  

传统方法中, 回调函数直接由异步操作执行, 使用 Task 可以自定义, 默认行为由 TaskScheduler 决定.  
async await 是简化版 Task 语法糖, Task 为我们提供了更加精细的粒度和更大的灵活性.  

## SynchronizationContextTaskScheduler

`SynchronizationContextTaskScheduler` 是 .NET 中一个特殊的任务调度器，它将任务的执行委托给 `SynchronizationContext`。这是连接 TPL (Task Parallel Library) 和传统线程模型的重要桥梁。

### 核心特性

1. **执行上下文传递**：
   - 将所有调度的任务通过 `SynchronizationContext.Post` 方法发送到目标上下文
   - 确保任务在创建调度器时的原始上下文中执行

2. **设计用途**：
   - 主要用于 UI 应用程序（WinForms, WPF）中保持线程亲和性
   - 使任务延续（continuations）自动回到 UI 线程执行

3. **实现细节**：
   - 继承自 `TaskScheduler` 抽象类
   - 最大并发度始终为 1（单线程执行）
   - 内部使用 `SynchronizationContext.Post` (异步发送) 而非 `Send`（同步发送）

### 典型使用场景

将默认的 SynchronizationContext 设置为 UI 线程的同步上下文, 这样所有使用到 SynchronizationContext 的地方都等于在 UI 线程执行.  

```csharp
// 在UI线程上获取当前同步上下文
var uiContext = SynchronizationContext.Current;

// 创建基于该上下文的调度器
var uiScheduler = new SynchronizationContextTaskScheduler(uiContext);

// 使用该调度器执行任务
Task.Factory.StartNew(() => 
{
    // 这段代码会在UI线程上执行
    button1.Text = "更新UI";
}, CancellationToken.None, TaskCreationOptions.None, uiScheduler);
```

### 与 async/await 的关系

async/await 模式默认会自动捕获 `SynchronizationContext`，其行为类似于使用 `SynchronizationContextTaskScheduler`：

```csharp
// 这两种方式在UI应用程序中等效：
var task1 = Task.Run(() => {}).ContinueWith(_ => 
{
    // 使用显式调度器
    button1.Text = "更新";
}, uiScheduler);

var task2 = Task.Run(async () => 
{
    await Task.Delay(100);
    // 自动捕获上下文
    button1.Text = "更新"; 
});
```

### 重要注意事项

1. **性能考虑**：
   - 不必要的上下文切换会影响性能
   - 在非UI代码中可使用 `ConfigureAwait(false)` 避免

2. **死锁风险**：
   - 如果在UI线程上同步等待任务完成（如 `.Result` 或 `.Wait()`），可能导致死锁

3. **替代方案**：
   - 对于简单UI更新，通常直接使用 `Control.Invoke`/`Dispatcher.Invoke` 更直观
   - 现代async/await模式已内置上下文捕获，通常不需要显式创建此调度器

### 内部实现简析

查看.NET参考源码，其主要逻辑是：

```csharp
protected override void QueueTask(Task task)
{
    _synchronizationContext.Post(_ => 
    {
        TryExecuteTask(task);
    }, null);
}
```

这种设计确保了所有任务都在原始同步上下文中序列化执行。  

## Unity3D 中 SynchronizationContext
在 Unity3D 中可以使用 async/await 语法糖, 自然也能够使用 Task. <span style='color:#e84118'>但是无法在 Task 中异步处理 UI 资源, 例如 GameObject, 会报异常.</span>  

<span style='color:#4cd137'>虽然 Task 异步操作无法使用 UI 资源, 但是同步之后的操作是可以正常使用 UI 资源的.</span>  


原因在于, Unity 使用了自定义的 UnitySynchronizationContext 替换了默认的 SynchronizationContext, 但是并没有替换默认的 ThreadTaskScheduler. 所以 Task 开启的异步操作依然是多线程, 但是同步操作被发送到 Unity UI 主线程中.   
  
```csharp
public class SynchronizationTest : MonoBehaviour
{
    int step = 0;
    // Start is called before the first frame update
    void Start()
    {
        Debug.Log($"Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        Test();
    }

    private void Update()
    {
        Debug.Log(step);
    }

    async void Test()
    {
        Debug.Log($"Test Start at {Thread.CurrentThread.ManagedThreadId}");
        await Task.Run(() => {
            Debug.Log($"Test Async at {Thread.CurrentThread.ManagedThreadId}"); 
            Thread.Sleep( 1000 );
            // exception: Find can only be called from main thread
            //var go = GameObject.Find("SynchronizationTest"); 
            step = 100;
        });
        //Debug.Log($"Test End at {Thread.CurrentThread.ManagedThreadId}");
        Debug.Log($"After Test End, Continue at {Thread.CurrentThread.ManagedThreadId}");
        var go = GameObject.Find("SynchronizationTest");
    }
}
```  

<span style='color:#4cd137'>多线程默认可以共享资源, 例如设置的共享变量 step, 很多时候多线程可以简单的使用共享变量进行通信</span>  

## Unity3D 正确的使用 async/await
从上面的分析可以得知, async/await 机制的异步调度依赖的是 await Task 这个异步操作对象, 编译器会将 async/await 代码转换成状态机, 并且将 MoveNext 注册为异步操作的回调函数.  

async/await 的驱动等同于 Task.Run().ContinueWith(asyncStateMachine.MoveNext()), 因为 Unity3D 将默认 SynchronizationContext 替换成为在主线程上执行的 UnitySynchronizationContext, 而 ContinueWith 默认使用 SynchronizationContext, 所以 Unity3D 中 await 之后同步操作语句可以正常访问 UI 主线程. 但是 await Task 依然是使用默认的 ThreadTaskScheduler, 所以依然是多线程异步, 无法访问 UI 主线程.  

伪代码如下:  
```csharp
void async UnityAsync()
{
    // 多线程执行异步调用, 网络或者 I/O 请求
    await Task.Run();

    // 多线程异步调用后, 可以操作 UI 线程上的 GameObject
    var go = GameObject.Find();
}
```  

<span style='color:#fbc531'>多线程相比于 Unity3D 自带协程有什么优势呢?</span>  
