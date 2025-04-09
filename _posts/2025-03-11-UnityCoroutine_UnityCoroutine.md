---
layout: post
title: Unity Coroutine (三) Unity3D 中的协程
subtitle: Unity3D 内置的协程模式
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
tags: Unity3D 游戏引擎
priority: 3
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine2.jpg)  

在 Unity3D 中，协程（Coroutine） 是一种特殊的函数，它可以在执行过程中暂停，并在稍后的时间点继续执行。协程通常用于处理需要分步执行的任务，例如延时、动画、异步加载等   

Unity3D 协程是异步编程的一种实现方式. 对于 协程常见的疑问:  
1. 协程是多线程吗? 协程并不是多线程, 但是与多线程都属于异步编程的实现
2. 协程是如何实现的? (猜测)在 Unity3D 中使用迭代器模式实现
3. 嵌套协程的行为? 嵌套协程是否返回这个协程.
4. Unity3D 中无法使用多线程吗? 实际上是可以的, 但是 Unity3D 中的很多对象只允许主线程访问.
5. Unity3D 脚本中为什么不能使用 async/await? 默认不可以, 原因和问题四差不多, 但是经过一些改造是可以的.

## Unity3D 协程的用法
既然 Unity3D 中的协程是单线程并且处于主线程, 那么协程阻塞一定会导致主线程阻塞.  
<span style='color:#fbc531'>那么协程中使用系统的 I/O 函数, 不是依然会阻塞主线程吗? 这似乎与协程的异步目的不符合</span>  
<span style='color:#4cd137'>协程的使用方法实际上和多线程同步相似, 不在主线程进行同步, 主线程只检查多线程执行状态(AsyncResult), 同样, 协程也只检查异步操作的状态.</span>  

下面是模拟 `Resources.LoadAsync` 异步读取的伪代码, 在协程中使用多线程, 并且检查多线程异步执行状态.  
```csharp
public class ResourceRequest : CustomYieldInstruction
{
    public Object asset { get; private set; }
    public float progress { get; private set; }
    public bool isDone { get; private set; }

    private string resourcePath;
    private System.Action<Object> onComplete;

    public ResourceRequest(string path)
    {
        resourcePath = path;
        progress = 0f;
        isDone = false;
        // 开启异步加载方法
        StartLoading();
    }

    private async void StartLoading()
    {
        try
        {
            // 使用异步方法加载资源
            asset = await LoadResourceFromDiskAsync(resourcePath);
            // 同步后操作, 设置完成状态
            isDone = true;
            // 执行回调方法
            onComplete?.Invoke(asset); // 触发完成回调
        }
        catch (System.Exception ex)
        {
            Debug.LogError("Error loading resource: " + ex.Message);
            isDone = true;
        }
    }

    // 异步加载方法
    private async Task<Object> LoadResourceFromDiskAsync(string path)
    {
        // 模拟从磁盘异步加载资源
        string fullPath = Path.Combine(Application.dataPath, "Resources", path + ".txt");

        // 使用 File.ReadAllTextAsync 异步读取文件内容
        string content = await File.ReadAllTextAsync(fullPath);

        // 读取完成后, 封装并返回
        return new TextAsset(content);
    }

    public override bool keepWaiting => !isDone; // 用于协程等待
}
```    


## 模拟 Unity3D 协程
协程的实现方式和使用方式多种多样. Unity3D 使用迭代器实现协程, Unity3D 中的协程理解为, 在每一帧特定时间点前进一步的方法就行. 协程执行等同于主线程执行.    

Unity3D 的协程的特性:  
1. 异步执行, 在主线程执行  
2. 按步骤执行, 每一个 yield return 分割了一轮执行操作
3. 嵌套协程, yield return Coroutine; 进入嵌套协程的执行流程

### 迭代器
CSharp 直接在语义层面支持迭代器, 简单的迭代器代码  

```csharp
IEnumerator GetNum()
{
    for(int i = 0;i < 10; i++)
    {
        yield return i;
    }
}

IEnumerator nums = GetNum();

// 等同于调用 MoveNext
// foreach (int num in nums)
// {
//     // num -> 0 ~ 9
// }

while(nums.MoveNext())
{
    // nums.Current -> 0 ~ 9
    nums.Current
}
```  

### 主线程异步执行
Unity3D 中使用迭代器实现协程, 并且协程就运行在主线程上, 那么不难猜到, 主线程中有管理协程运行的调度器, 具体到迭代器上, 就是遍历 Coroutines 队列调用每一项的 `MoveNext()` 方法  

  
```csharp
// 在某个特定的时间段
foreach(IEnumerator coroutine in coroutines)
{
    coroutine.MoveNext();
}
```  

Unity 中内置了各种类型的协程, 在 Unity 引擎不同的生命周期进行调度, 例如   
+ yield return null. 等到下一帧 Update 后继续执行下一步
+ yield return new WaitForEndOfFrame. 等待帧渲染完毕执行下一步
+ yield return new WaitForFixedUpdate. 等待FixedUpdate完毕执行下一步


### 嵌套协程
嵌套协程有两种容易让人混淆的形式:  
1. 在协程中开启一个新的协程
2. yield return new Coroutine  

第一种, 类似于在函数中开启一个异步而无需等待同步  
```csharp
IEnumerator GetNum()
{
    for(int i = 0;i < 10; i++)
    {
        Debug.Log(i);
        // 直接开启十个异步操作
        StartCoroutine(DoSomeThingAsync());
    }
    Debug.Log("GetNum End");
    yield return null;
}

IEnumerator DoSomeThingAsync()
{
    // do DoSomeThing
    Debug.Log("Do Something Async Start");
    yield return null;
    Debug.Log("Do Something Async End");
}
```  
显示顺序:  
1.  i 值(0-9) 和 Do Something Async Start
2.  GetNum End. 1 和 2 都是发生在同一帧中
3.  新的一帧, 十个 Do Something Async End
没有同步的协程调用, 协程按照原先顺序该怎么执行还是怎么执行  


第二种, 在函数中开启一个异步操作, 并且需要等待同步  
```csharp
   int FrameCount = 0;
   // Start is called before the first frame update
   void Start()
   {
       StartCoroutine(GetNum());
   }

   // Update is called once per frame
   void Update()
   {
       FrameCount++;
   }

   IEnumerator GetNum()
   {
       for(int i = 0; i < 10; i++)
       {
           Debug.Log($"FrameCount {FrameCount} : GetNum {i}");
           //StartCoroutine(DoSomethingAsync());
           //yield return null;
           yield return StartCoroutine(DoSomethingAsync());

           //StartCoroutine(DoSomethingAsync());
       }

       Debug.Log($"FrameCount {FrameCount} : GetNum End");
       yield return null;
   }

   IEnumerator DoSomethingAsync()
   {
       Debug.Log($"FrameCount {FrameCount} : Do Something Async Start");
       yield return null;
       Debug.Log($"FrameCount {FrameCount} : Do Something Async End");
   }
```  

显示顺序:  
1. FrameCount 0 : GetNum 0
2. FrameCount 0 : Do Something Async Start
3. FrameCount 2 : Do Something Async End
4. FrameCount 2 : GetNum 2
5. FrameCount 2 : Do Something Async Start
6. FrameCount 3 : Do Something Async End
7. FrameCount 3 : GetNum 3
8. FrameCount 3 : Do Something Start
9. 重复流程

显然, GetNum 等待 DoSomethingAsync 执行完毕才会开启下一步操作, 嵌套协程的同步行为.  

## 有栈协程
自定义类模仿 Unity3D 协程, 主要麻烦的地方就是嵌套栈的实现.  

```csharp
        public class StackCoroutine : IEnumerator
        {
            private Stack<IEnumerator> m_EnumeratorStack = new();
            private bool m_IsStop;

            public bool IsStop => m_IsStop;
            public string Name { get; set; }

            public StackCoroutine(IEnumerator enumerator)
            {
                m_IsStop = false;
                Name = string.Empty;
                m_EnumeratorStack.Push(enumerator);
            }

            // 内部迭代器栈容量为0, 说明执行完毕, Current设置为Null, 否则返回栈顶迭代器
            public object Current
            {
                get
                {
                    if (m_EnumeratorStack.Count == 0)
                    {
                        return null;
                    }
                    else
                    {
                        return m_EnumeratorStack.Peek();
                    }
                }
            }

            public bool MoveNext()
            {
                // 如果内部迭代器栈为空, 表示没有需要迭代的操作
                if (m_EnumeratorStack.Count == 0)
                {
                    // 执行完毕, 协程退出
                    return false;
                }

                // 执行栈顶的协程
                IEnumerator curEnumertor = m_EnumeratorStack.Peek();

                if(curEnumertor.MoveNext())
                {
                    // 判断迭代器的返回值是否是嵌套协程
                    if (curEnumertor.Current is StackCoroutine)
                    {
                        // 优先判断 StackCoroutine
                        // 嵌套 StackCoroutine
                        // 注意, 嵌套的 StackCoroutine 表示一个新的异步协程, 它的运行和当前协程毫无关系
                        // 当前协程需要做的, 仅仅是等待嵌套协程完成
                        // 返回值是一个独立的Coroutine
                        // 说明新开启了一个嵌套 Coroutine
                        // 类似于多线程调用, async/await机制, 返回这个类型, 说明这里需要等待同步
                        // 创建一个类似YieldInstrcution机制, 包装一个迭代器等待异步Coroutine执行完毕.
                        // 虽然MoveNext()返回false就是执行完毕, 但是本身是执行一次迭代.
                        // 因此通过Coroutine的Current来判断是否执行完毕. 

                        // error 闭包 curEnumerator 错误
                        //WaitWhile wait = new WaitWhile(() => curEnumertor.Current != null);

                        WatiForStackCoroutine wait = new WatiForStackCoroutine((StackCoroutine)curEnumertor.Current);
                        // 将等待操作入栈
                        m_EnumeratorStack.Push(wait);
                    }
                    else if (curEnumertor.Current is IEnumerator)
                    {
                        // IEnumerator and StackCoroutine
                        // yield return IEnumerator 和 yield return StackCoroutine 的区别
                        m_EnumeratorStack.Push((IEnumerator)curEnumertor.Current);
                    }
                    // 
                    return true;
                }
                else
                {
                    // 当前协程执行完毕, 出栈
                    m_EnumeratorStack.Pop();

                    return m_EnumeratorStack.Count > 0;
                }
            }

            public void Stop()
            {
                m_IsStop = true;
            }

            public void Reset()
            {
                m_EnumeratorStack.Clear();
                m_IsStop = false;
            }
        }

        // while(predicate()){ wait; }
        public class WaitWhile : IEnumerator
        {
            Func<bool> m_Predicate;

            public object Current => null;


            public WaitWhile(Func<bool> predication)
            {
                this.m_Predicate = predication;
            }

            public bool MoveNext()
            {
                // 只要条件成立, 就一直等待
                return m_Predicate();
            }

            public void Reset()
            {
                m_Predicate = null;
            }
        }

        public class WatiForStackCoroutine : IEnumerator
        {
            private StackCoroutine m_Coroutine;
            private WaitWhile m_WaitWhile;
            public object Current => null;

            public WatiForStackCoroutine(StackCoroutine stackCoroutine)
            {
                m_Coroutine = stackCoroutine;
                m_WaitWhile = new WaitWhile(() => m_Coroutine.Current != null);
            }

            public bool MoveNext()
            {
                return m_WaitWhile.MoveNext();
            }

            public void Reset()
            {
                m_Coroutine = null;
                m_WaitWhile = null;
            }
        }

        // 管理有栈协程的类
        public static class CoroutineDict
        {
            public static Dictionary<string, StackCoroutine> coroutineDict = new Dictionary<string, StackCoroutine>();

            static string prefix = "MyCoroutine";
            static int m_Count;
            public static StackCoroutine StartCoroutine(IEnumerator enumerator)
            {
                string prefixName = prefix + m_Count.ToString();
                return StartCoroutine(prefixName, enumerator);
            }

            public static StackCoroutine StartCoroutine(string name, IEnumerator enumerator)
            {
                if(coroutineDict.ContainsKey(name))
                {
                    throw new Exception($"Coroutine {name} is already Exists, should set different name.");
                }

                var coroutine = new StackCoroutine(enumerator);

                coroutineDict.Add(name, coroutine);
                coroutine.Name = name;
                return coroutine;
            }

            public static void StopCoroutine(string name)
            {
                if (coroutineDict.ContainsKey(name))
                {
                    coroutineDict[name].Stop();
                }
            }

            public static void Loop()
            {
                List<string> finishedCoroutine = new List<string>();
                string[] keys = coroutineDict.Keys.ToArray();
                for (int i = 0; i < keys.Length; i++)
                {
                    var curCoroutine = coroutineDict[keys[i]];

                    if (curCoroutine.IsStop)
                    {
                        continue;
                    }

                    if(!curCoroutine.MoveNext())
                    {
                        // cur coroutine 执行结束, 需要从待执行列表中移除, 先加入结束列表
                        finishedCoroutine.Add(curCoroutine.Name);
                    }

                }

                // 从待执行列表中移除已经结束的协程
                foreach (string finished in finishedCoroutine)
                {
                    coroutineDict.Remove(finished);
                }
            }
        }

        static int loopCount = 0;
        static void Main(string[] args)
        {
            Console.WriteLine("Coroutine Loop");
            var testCoroutine = CoroutineDict.StartCoroutine("Test", Test());
            //var nestCoroutine = CoroutineDict.StartCoroutine("NestedTest", NestedTest());
            //var nestedCoroutine = CoroutineDict.StartCoroutine("Test2", Test2());

            // 模拟调度器
            while (true)
            {
                CoroutineDict.Loop();

                Thread.Sleep(200);
                loopCount++;

            }
        }

        static IEnumerator Test()
        {
            Console.WriteLine($"Loop Count {loopCount}: Test Start");
            yield return CoroutineDict.StartCoroutine(NestedTest());
            Console.WriteLine($"Loop Count {loopCount}: Test End");
        }

        static IEnumerator NestedTest()
        {
            Console.WriteLine($"Loop Count {loopCount}: Nested Test Start.");
            yield return 1;
            yield return 2;
            yield return 3;
            Console.WriteLine($"Loop Count {loopCount}: Nested Test End.");
        }

        static IEnumerator Test2()
        {
            Console.WriteLine($"Loop Count {loopCount}: Test2 Start");
            yield return NestedTest();
            Console.WriteLine($"Loop Count {loopCount}: Test2 End");
        }
```  

### 问题
