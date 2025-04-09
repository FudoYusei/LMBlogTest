---
layout: post
title: Unity Coroutine (六) 从零实现 Unity3D 协程模式
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
priority: 6
---

![banner](/assets/images/banners/UnityCoroutine/WhatsCoroutine2.jpg)  

自定义类模仿 Unity3D 协程的行为:   
1. 主线程调度协程异步运行.
2. 嵌套协程同步运行.

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

## 细节问题
Unity3D 开启协程会立即在同一帧推进一次协程. <span style='color:#4cd137'>个人猜测原因如下, 因为 IEnumerator 迭代器模式由编译器标准化生成, 必然至少有一个 yield return, 也就是至少可以调用一次 MoveNext</span>  

问题就在于这个 MoveNext 不是一定需要等到下一次调度器统一执行 MoveNext, 而是立即执行一次 MoveNext.  

个人猜测, Unity3D 中的协程应该是使用类似于 scheduler.Schedule(coroutine, ScheduleMode.Immediate) 或者 scheduler.Schedule(coroutine, ScheduleMode.Update) 来进行调度.  

Coroutine 会在 MoveNext 方法中, 不断的向调度器提交操作, 例如:  
1. 返回一个嵌套 IEnumerator, 需要立即调度执行一次 MoveNext. 嵌套协程不需要, 因为在开启协程时已经执行过一次.  
2. 当嵌套协程或者 IEnumerator 执行完毕(同步操作完毕), 立即调度执行一次主协程.

以后有机会, 再尝试这种更加复杂但自由的结构.  