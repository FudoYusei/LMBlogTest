---
layout: post
title: Algorithm (四) 栈
subtitle: 先进后出的特性
author: LM
categories: 算法
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
excerpt_image: /assets/images/Algorithm/Algorithm_Overview.png
tags: 算法
priority: 4
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 栈
栈在算法中的特性:  
1. 先进后出, 极其简单的特性, 却是编程的基石

栈在算法中的用法:  
1. 递归(隐式利用函数栈存储临时变量)
2. 需要操作相对于当前元素时间上最近
3. 需要操作相对于当前元素距离上最近


## 有效的括号
[有效的括号](https://leetcode.cn/problems/valid-parentheses/description/)  

### 栈
思路如下:  
1. 左括号入栈
2. 每个右括号消除距离自己最近的匹配左括号, 也就是出栈
3. 若不能匹配, 则括号无效

```csharp
    Dictionary<char, char> dict = new Dictionary<char, char>()
    {
        {')', '('},
        {']', '['},
        {'}', '{'}
    };


    public bool IsValid(string s)
    {
        Stack<char> stack = new Stack<char>();
        for(int i = 0;i < s.Length;i++)
        {
            if(!dict.ContainsKey(s[i]))
            {
                // 左括号入栈
                stack.Push(s[i]);
            }else{
                // 右括号匹配栈顶
                if(stack.Count==0)
                {
                    // 无法匹配, 返回 false
                    return false;
                }else{
                    var top = stack.Pop();

                    if(top != dict[s[i]])
                    {
                        // 匹配失败, 返回false
                        return false;
                    }
                }
            }
        }

        return stack.Count == 0;
    }
```  

### DFS  
DFS 解法就是另一种形式的栈, <span style='color:#4cd137'>函数栈代替外在的辅助栈</span>  
1. 入栈的变量就是函数局部变量: 使用局部变量记录每一层的字符串
2. 同时左括号入栈, 也就是递归函数调用点.
3. 右括号出栈, 对应递归函数返回点是右括号.
4. 递归函数返回右字符串, 使用局部变量匹配现在指向的右括号.
5. DFS 函数表示获取当前 Step 所在位置两个括号内的字符串.

```csharp
       Dictionary<string, string> dict = new Dictionary<string, string>()
   {
       {")", "("},
       {"]", "["},
       {"}", "{"}
   };
       public bool IsValid(string s)
       {
           return DFS(s) == string.Empty;
       }

       int step = 0;

       /// <summary>
       /// 寻找当前 Step 右侧的右括号并返回
       /// 有点像后序遍历
       /// </summary>
       /// <param name="s"></param>
       /// <returns></returns>
       public string DFS(string s)
       {
           // 1. 递归返回条件
           if (step >= s.Length)
           {
               // 到达末尾
               return string.Empty;
           }

           if (dict.ContainsKey(s[step].ToString()))
           {
               // 右括号
               return s[step].ToString();
           }

           // 2. 如果是左括号, 递归下降

           // 2.1 记录局部变量
           string left = s[step].ToString();

           // 2.2 递归函数调用
           step++;
           string right = DFS(s);
           
           if(dict.ContainsKey(right) && dict[right] == left)
           {
               // 匹配成功, 说明当前左括号可以消除
               left = string.Empty;
           }

           // 接着递归
           step++;
           return left+DFS(s);
       }
```    


## 字符串解码
[字符串解码](https://leetcode.cn/problems/decode-string/description/) 典型的应用栈先进后出的特性.  

### 栈解法
核心思路: 将字符和数字当作 TOKEN, [ 看作入栈操作, ] 看作出栈操作.  
使用两个辅助栈, 一个记录字符, 一个记录数字.   

解题要点:  
1. [ 字符作为入栈的起点, 将当前拼接好的字符串与数字作为历史入栈.
2. ] 字符作为出栈的时机, 返回 [...] 之间的字符串
3. 从栈中取出数字作为重复次数, 重复拼接[...]返回的字符串
4. 从栈中取出历史字符串拼接重复字符串
5. 函数主流程是拼接字符串

```csharp
        public string DecodeString(string s)
        {
            // 1. 函数局部变量
            StringBuilder res = new StringBuilder();
            int multi = 0;

            // 2. 函数栈
            Stack<int> intStack = new Stack<int>();
            Stack<string> strStack = new Stack<string>();

            for(int i = 0; i < s.Length; i++) {
                if (s[i] - 'a' >= 0 && s[i] - 'a' <= 25)
                {
                    // 字符串拼接
                    res.Append(s[i]);
                }else if (s[i] - '0' >= 0 && s[i] - '0' <= 9)
                {
                    // 数字
                    multi = multi * 10 + (s[i] - '0');
                }else if (s[i] == '[')
                {
                    // 1. 模拟函数调用, 入栈函数局部变量
                    // abc2[dd] 入栈 abc 和 2 两个局部变量
                    intStack.Push(multi);
                    strStack.Push(res.ToString());

                    // 进入新的递归函数后, 局部变量重置
                    multi = 0;
                    res.Clear();
                }else if (s[i] == ']')
                {
                    // 2. 模拟函数返回, 返回值就是当前拼接字符串 res
                    // abc2[dd] -> retStr = dd
                    StringBuilder retStr = new StringBuilder(res.ToString());
                    // 3. 因为函数返回, 所以修复函数局部变量, 出栈
                    // 3.1 restore int
                    // abc2[dd] -> multi = 2, res = abc
                    multi = intStack.Pop();
                    res.Clear();
                    res.Append(strStack.Pop());

                    // 4. 继续递归函数返回后的操作(后序遍历)
                    // 现在的局部变量已经是当前层
                    // 4.1 repeat string
                    // abc2[dd] -> repeatStr = dddd
                    StringBuilder repeatStr = new StringBuilder();
                    for(int repeat = 0; repeat < multi; repeat++)
                    {
                        repeatStr.Append(retStr);
                    }
                    // 记住: multi 用完, 清零
                    multi = 0;

                    // 4.2 当前字符串与重复字符串拼接
                    // abc2[dd] -> res = abcdddd
                    res.Append(repeatStr.ToString());
                    
                }
            }

            return res.ToString();
        }
```  

### 递归解法
1. 入栈的两个参数, 数字和历史字符串, 所以这两个一定是函数的局部变量, 这样才会保存在函数栈中.
2. [ 符号是入栈的起点, 因此也是递归调用的时间点
3. ] 符号是出栈的时机, 因此也是递归方法返回的时机
4. 函数的主体流程是拼接字符串, 返回值也是拼接好的字符串.
5. 当接收到递归方法的返回值后, 需要使用数字参数和历史字符串拼接成当前层的字符串. (类似后序遍历?)  

```csharp
        public string DecodeString(string s)
        {
            return DFS(s);
        }

        // 必需全局变量
        int start = 0;
        /// <summary>
        /// 
        /// </summary>
        /// <param name="s"></param>
        /// <param name="start"></param>
        /// <returns></returns>
        public string DFS(string s)
        {
            // 1. 函数局部变量
            int multi = 0;
            StringBuilder res = new StringBuilder();


            while(start < s.Length)
            {
                if (s[start] - 'a' >=0 && s[start] - 'a' <= 25)
                {
                    // 字母
                    res.Append(s[start]);
                }else if (s[start] - '0' >= 0 && s[start] -'0' <= 9) {
                    // 数字
                    multi = multi * 10 + (s[start] - '0');
                }
                else if (s[start] == '[')
                {
                    // 下一层递归
                    start++;
                    StringBuilder retStr = new StringBuilder(DFS(s));

                    // 后序遍历操作
                    StringBuilder repeatStr = new StringBuilder();
                    // 1. 拼接重复字符串
                    for (int repeatTime = 0; repeatTime < multi; repeatTime++)
                    {
                        repeatStr.Append(retStr);
                    }
                    res.Append(repeatStr.ToString());
                    multi = 0;
                }else if (s[start] == ']')
                {
                    // 递归返回
                    return res.ToString();
                }
                start++;
            }

            // 遍历完整个字符串, 返回最终结果
            return res.ToString();
        }
```  

可以看到栈和递归可以互相转换, 递归本身就是利用函数栈. 通常来说, 递归的逻辑性更强, 使用栈需要清晰的理解操作的上下层关系, 非常容易混淆.  

  
[协程出现的原因]: https://www.zhihu.com/question/50185085/answer/183463734  
[协程与多线程]: https://blog.csdn.net/weixin_44575037/article/details/105513014    
