---
layout: post
title: Unity3D Coroutine (七) Task解密
subtitle: Task 和 TaskScheduler
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
priority: 7
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine2.jpg)  

Charp 中的 Task 和 async/await 都是语法糖, 用于简单快捷的实现协程. Task 对象表示一个可等待的异步任务, 这篇文章深入研究 Task 背后的机制.  

---  

## Unity3D 中的 Task
Unity3D 中能够使用 Task 以及 async/await 吗? 在 Unity3D 中使用 Task 与使用自带的协程有什么区别?  

```csharp
public class SynchronizationTest : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        Debug.Log($"Main Thread ID: {Thread.CurrentThread.ManagedThreadId}");
        Test();
    }

    async void Test()
    {
        Debug.Log($"Test Start at {Thread.CurrentThread.ManagedThreadId}");
        await Task.Run(() => {
            Debug.Log($"Test Async at {Thread.CurrentThread.ManagedThreadId}"); 
            Thread.Sleep( 1000 ); 
            // exception: Find can only be called from main thread
            //var go = GameObject.Find("SynchronizationTest"); 
        });
        //Debug.Log($"Test End at {Thread.CurrentThread.ManagedThreadId}");
        Debug.Log($"After Test End, Continue at {Thread.CurrentThread.ManagedThreadId}");
        var go = GameObject.Find("SynchronizationTest");
    }
}
```  
返回结果:  
Main Thread ID: 1
Test Start at 1
Test Async at 62
After Test End, Continue at 1

如果取消 Task.Run 里面的注释会得到异常:  
Test Async at 62
UnityException: Find can only be called from the main thread.

Unity3D 中可以使用 async/await. <span style='color:#4cd137'>并且 U3D 中特殊的点在于, await 同步后操作依然会在主线程上执行.</span> 非 U3D 中, 默认 await 同步后的操作会可能会调度一个线程执行.  

<span style='color:#4cd137'>使用 Unity3D 自带的协程可以随意操作 Engine 中的对象, 因为 Unity Coroutine 实际上都是运行在主线程(UI线程) 上的.</span>  

从 Task 与 U3D 协程的区别开始讲解 Task 与协程背后的机制.  

---  

## 没有Task之前的异步和回调
https://www.cnblogs.com/xsj1989/p/7831694.html  

多线程之间的同步通信, 需要使用共享的事件机制, 当需要同步的时候, 一方订阅事件并被挂起, 等待另一方触发事件.  

Task实际上隐藏了这些麻烦的细节, 让我们简单就能完成异步调用, 回调和主线程同步.   

---  

## Task 的常见用法  
https://zhuanlan.zhihu.com/p/646609116    

主线程上基本不可能使用async await. 因为await之后的语句实际上就是回调函数.  
它们变成回调函数, 那么主线程等同于直接结束了  

```c
        public static async void DoAsync()
        {
            await Task.Delay(2000);
            Console.WriteLine("Im Callback");
        }

        static void Main(string[] args)
        {
            DoAsync();
        }
```  
主线程结束, 程序退出, 其他的子线程跟着结束  

Task采取了更加复杂的机制来保证多线程任务的各种特性.  

比如, Task可以使用ContinueWith随意添加回调方法  
Task的回调和上面的回调不同.  
Task的回调方法, 更像是后驱方法, 它本身也是Task, 跟随着Task本身执行完毕. 接着执行后续的Task  

<span style='color:#fbc531'>异步方法的使用与普通函数差不多, 但是会发觉调用异步方法就脱离了调用者的控制, 那么是谁来驱动异步方法执行? 又是谁在异步方法执行完毕之后来执行后续的同步操作呢?</span>  

---  

##  Task 驱动者 - ThreadTaskScheduler  
<span style='color:#4cd137'>TaskScheduler 用于进行异步方法本体的调度, 相对的 SynchronizationContextScheduler 用于回调函数的调度.</span>  

极其简单的自定义TaskScheduler  

```c 
        public static async void DoAsync()
        {
            var task = Task.Factory.StartNew(() =>
            {
                Thread.Sleep(2000);
                Console.WriteLine("Do Async Completed");
            }, CancellationToken.None, TaskCreationOptions.None, new perThreadTaskScheduler());
            //await task;
            task.ContinueWith(
                 (t) => {PrintCallback(); }, CancellationToken.None, TaskContinuationOptions.None, new perThreadTaskScheduler());
        }

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
```  

核心方法是QueueTask. 开启一个新的线程, 线程的执行主体是委托 TryExecuteTask(task)   

TryExecuteTask(Task task) -> task.ExecuteEntry();  

// 执行task的最外层入口
ExecuteEntry() -> InnerInvoke()  
InnerInvoke() 就是执行传入Task的委托  

执行完InnerInvoke()之后, 会有一个Finish(true)  
Finish(true) 就是回调调用的地方  

### ThreadPoolWorkQueue  
如果为Task 新开辟一个 Thread, Thread会进入ThreadPoolWorkQueue  

线程有着自主运行管理线程队列的存在(通常是操作系统). 我们只需要将委托放入队列, 它会在某个时间下执行队列. 如果队列中的元素是Task 会调用ExecuteFromThreadPool()  

ThreadPoolWorkQueue.cs
```c
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        private static void DispatchWorkItem(object workItem, Thread currentThread)
        {
            if (workItem is Task task)
            {
                task.ExecuteFromThreadPool(currentThread);
            }
            else
            {
                Debug.Assert(workItem is IThreadPoolWorkItem);
                Unsafe.As<IThreadPoolWorkItem>(workItem).Execute();
            }
        }
```  

---  

## Task 驱动者 - SynchronizationContextTaskScheduler  
如果牵扯到Thread, 因为Thread复杂的机制, 导致无法清晰的看出Task的执行脉络  
那么自定义一个SynchronizationContextTaskScheduler 就更容易看清了  

常见于带有UI元素的应用程序, 比如各类游戏引擎或者WPF这种桌面  
UI元素只能在UI线程(主线程操作).  
当我们给异步操作传入回调函数时.  
例如, Socket异步连接, 此时播放正在连接动画, 连接完成后同步, 关闭连接动画, 开启下一步操作  
如果按照默认形式, 回调函数会自动开启一个线程进行异步操作. 但是非UI线程(主线程)无法调用UI元素.  

一般拥有这种特性的框架, 都会默认实现一个SynchronizationContext  

```c
var t = Task.Run(()=>{DoAsync()};);
t.ContinueWith((t)=>{callback();}, , ,TaskScheduler.FromCurrentSynchronzationContext);
```  
现在Callback()就可以在主线程上调用  

同时, 因为async/await默认使用SynchronizationContext  
  
<span style='color:#fbc531'>async/await 在Unity中使用的话, 默认Callback()在主线程调用吗?</span>  

SynchronzationContext 的作用是异步操作之间用来通信:  
每一个线程都有一个SynchronizationContext, 异步线程执行完毕, 会根据自身的 SynchronizationContext 来通知对象执行完毕的消息   
主线程会在特定的地方开始MessageLoop  

https://blog.csdn.net/u010476739/article/details/118738041  


<font color=#4cd137>再次注意, QueueTask()并不管参数是什么异步同步还是回调方法, 它只负责将函数参数传递到它能执行的地方</font>  
因为这个SynchronizationContextTaskScheduler 是默认实现, 还是使用ThreadPool进行同步  
ConsoleApp本来就是默认使用ThreadPoolTaskScheduler  

怎么让Task自己异步实现, 而回调函数进行Post呢?  
<font color=#4cd137>SynchronizationContext的使用场景和TaskScheduler本身就是不同的</font>  
也就是说SendOrPost本身就是回调函数, 或者要同步的函数   
例如常见的使用场景是 async await自动生成代码, await之后的代码都默认使用 TaskScheduler.FromCurrentSynchronizationContext  

Unity自己实现的UnitySynchronizationContext  
https://github.com/Unity-Technologies/UnityCsReference/blob/master/Runtime/Export/Scripting/UnitySynchronizationContext.cs#L27  

---  

## TaskScheduler  
https://www.cnblogs.com/cdaniu/p/15845424.html  

TaskScheduler只是负责将送进来的任务调度到对应的地方执行.  
至于进来的任务是异步方法, 同步方法还是回调方法, 它不会去分辨  

这也是为什么自定义的Scheduler可以简单的应用于Task  
Task和TaskScheduler完全不是一个规模的复杂度.  

Task最外层的API task.ExecuteEntry() 或者同样的 ExecuteEntryUnsafe()  
这个方法执行传入Task的委托, 并且触发当前task的后续ContinuationTask的执行  

ThreadPoolTaskScheduler 中将task加入到 WorkQueue中  
ThreadPool管理WorkQueue, 并且会进行Dispatch  

对于Task的处理, 最终也是调用ExecuteEntryUnsafe()  
![alt text](\assets\images\UnityCoroutineTaskScheduler\image-1.png)  


TaskScheduler中的方法, TryExecuteTask   
![alt text](\assets\images\UnityCoroutineTaskScheduler\image-2.png)  

<font color=#4cd137>需要明确的是:  task.ExecuteEntry() 并没有神神奇的, 就是普通的方法执行</font>  
并不存在task.ExecuteEntry() 就是为了异步设计的  

所有的方法都是一个入口, 哪个线程执行这个入口, 这个方法就是哪个线程上执行的   

<font color=#4cd137>SynchoronizationContextTaskScheduler实际上就是将 task.ExecuteEntry()和 task 传递给我们想要同步的线程. 这样等于回调方法就同步到了想要的线程上</font>  

---  

## FromCurrentSynchronizationContext()  
这个方法返回一个SynchronizationContextTaskScheduler类  
这个类使用SynchronizationContext.Current作为消息中心    
因此, 自定义同步机制的, 诸如Unity, 实现了自己的UnitySynchronizationContext  

将我们想要同步的委托和参数提交到消息中心, 主线程会开启一个MessageLoop, 执行队列中的委托  

![alt text](\assets\images\UnityCoroutineTaskScheduler\image-4.png)  

核心就是将task.ExecuteEntry发送到m_SynchronizationContext中  

1. m_SynchronizationContext 相当于消息中心, 因此我们想要同步的主线程要有一个MessageLoop, 执行队列中的消息
2. 直接使用Task.Run和ContinueWith默认都是使用TaskScheduler.Default, 因此定义SynchoronizationContext不会影响默认的异步操作行为
3. 但是async await await语句之后的回调方法 会受到 SynchronizationContext影响, 他会使用SychronizationContextTaskScheduler来调度生成的回调方法   
4. 大概是对SychronziationContext.Current进行判断? 如果不为null, 就使用 SychronizationContextTaskScheduler来调度生成的回调方法. 这样async await的语句确实更加正常, 它本身就是同步的代码结构, 回调方法同步在调用async的线程上更加合理  

---  

## 谁负责驱动回调函数
首先, 已知async await生成的状态机, 将MoveNext()委托传入 awaiter.OnCompleted(Action continuation) 中  
使用最常见的Task, await task 等于 task.GetAwaiter().OnCompleted(MoveNext);  

![alt text](\assets\images\UnityCoroutineTaskScheduler\image-5.png)  

如果SynchronizationContext.Current不为空并且是实现了SynchronizationContext的扩展类  
就创建一个 SynchronizationContextAwaitTaskContinuation() 实例并加入task的m_ContinuationObject  
![alt text](\assets\images\UnityCoroutineTaskScheduler\image-6.png)  

AwaitTaskContinuation的核心方法 Run  
同步方法核心就是 SyncContext.Post()  
无论哪个线程调用 SyncContext.Post() 都可以将指定的方法同步给我们想要的线程  

![alt text](\assets\images\UnityCoroutineTaskScheduler\image-7.png)  

---  

## 总结
<span style='color:#4cd137'>协程从某种角度理解就是一步一步的执行流程, 某些步骤是异步操作, 某些步骤之间需要同步.</span>  

编译器会将 async/await 转换成统一的迭代器代码, 而迭代器最核心的函数就是 `MoveNext()`, 也就是一步一步的执行流程. 调用一次 `MoveNext()` 就是推进到下一步.  

Task 和 async/await 机制需要分开理解. Task 是异步操作对象, 抽象了使用异步方法的流程. Task 对象也没有什么神奇之处, 它包含异步操作主体, 与自身的注册回调函数机制.  
<span style='color:#4cd137'>await 之后必需跟随一个异步对象, 将自身状态机的 MoveNext() 注册为异步对象的执行完毕的回调方法.</span> 这样当异步操作完成之后, 会自动调用回调方法, 也就是注册的 MoveNext() 方法, 这样就完成了等待异步操作完成之后的下一步同步操作.  
