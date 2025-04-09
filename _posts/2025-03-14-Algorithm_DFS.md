---
layout: post
title: Algorithm (六) 深度优先搜索
subtitle: 常用的路径算法
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
priority: 6
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 深度优先搜索
深度优先搜索应该是最符合人类思考方式的一种方法, 也是一种递归形式.  
<span style='color:#4cd137'>深度优先搜索也没有什么神奇的地方, 首先需要定义能够覆盖所有情况的状态参数, 深度优先只是能够遍历所有情况的方法之一</span>  
深度优先基本都使用递归来实现.  

## 单词搜索
[单词搜索](https://leetcode.cn/problems/word-search/description/)  

假设这是一个 2x2 格子, 那么怎么定义一个能够覆盖所有路径的状态参数呢?  
很简单, 就是定义二维索引 [row, col]  
题意限定移动方向且不能重复已经路过的格子, 所以其中一条路径就是 [0, 0] -> [0 ,1] -> [1, 0] -> [1, 1]  

DFS 方法:  
1. 确定递归返回条件: 当前格子的值不匹配返回 false, 以及当路径到达末尾时匹配, 返回true
2. 设置当前格子的局部变量: visited[row][col] = true
3. 确定下一层递归的状态参数: newRow 和 newCol  

回溯法: 在递归方法中重置一个外部状态, 例如外部的 Visited 数组.  
<span style='color:#4cd137'>实际上, 这个外部状态使用外部数组的唯一原因就是节省内存. 因为完全可以将外部数组作为状态参数传参. 但是 CSharp 无法直接将数组作为值传参. 实际上后面会看到使用类似 BitMap 的技术来替换 Visited 数组</span>  

```csharp
       bool[] visited;
       
       public bool Exist(char[][] board, string word)
       {
           int rows = board.Length;
           int cols = board[0].Length;
           visited = new bool[rows * cols];

           for(int i = 0; i < rows; i++)
           {
               for(int j = 0; j < cols; j++)
               {
                   if (DFS(board, word, 0, i, j))
                   {
                       return true;
                   }
               }
           }
           return false;
       }

       int[] dRow = { 0, 0, 1, -1 };
       int[] dCol = { 1, -1, 0, 0 };
       public bool DFS(char[][] board, string word, int step, int row, int col)
       {
           // 1. 递归返回条件
           if(step == word.Length-1)
           {
               return board[row][col] == word[step];
           }

           if (word[step] != board[row][col])
           {
               // 字母不匹配
               return false;
           }

           // 2. 设置当前层的状态
           visited[row * board[0].Length + col] = true;

           // 3. 向下递归
           bool res = false;
           for(int i = 0; i < dRow.Length; i++)
           {
               int newRow = dRow[i] + row;
               int newCol = dCol[i] + col;

               if (newRow >= 0 && newRow < board.Length && newCol >= 0 && newCol < board[0].Length
                   && !visited[newRow * board[0].Length + newCol])
               {
                   // 新索引属于合法索引并且未被访问过
                   res |= DFS(board, word, step+1, newRow, newCol);
               }
           }

           // 4. 状态重置
           visited[row * board[0].Length + col] = false;

           return res;
       }
```  

## 单词搜索II
[单词搜索II](https://leetcode.cn/problems/word-search-ii/description/)  

单词搜索II 不同点是多单词匹配, 需要多个单词匹配, 也就是匹配操作需要得到升级  
多个英文单词搜索, 自然就可以使用前缀树.   

```csharp
    public class WordSearchIISolution
    {
        /// <summary>
        /// 前缀树节点
        /// </summary>
        public class TrieNode
        {
            public string word;
            public TrieNode[] Children;

            public TrieNode()
            {
                word = string.Empty;
                Children = new TrieNode[26];
            }
        }

        // 行数量
        int RowNum;
        // 列数量
        int ColNum;
        // 访问标记数组
        bool[] Visited;
        // 答案
        IList<string> ans;
        public IList<string> FindWords(char[][] board, string[] words)
        {
            RowNum = board.Length;
            ColNum = board[0].Length;
            Visited = new bool[RowNum * ColNum];
            ans = new List<string>();

            // 使用单词构建 Trie tree 
            TrieNode root = new TrieNode();
            for (int i = 0; i < words.Length; i++)
            {
                TrieNode cur = root;
                foreach (char ch in words[i])
                {
                    if (cur.Children[ch-'a'] == null)
                    {
                        cur.Children[ch-'a'] = new TrieNode();
                    }
                    cur = cur.Children[ch-'a'];
                }
                cur.word = words[i];
            }

            for(int i = 0; i < RowNum; i++)
            {
                for(int j = 0; j < ColNum; j++)
                {
                    // 以 board 中每个单字开头
                    DFS(board, root, i, j);
                }
            }

            return ans;
        }

        // 方向
        int[] dRow = { 0, 0, 1, -1 };
        int[] dCol = { 1, -1, 0, 0 };

        public void DFS(char[][] board, TrieNode node, int row, int col)
        {
            // 1. 递归返回条件
            TrieNode cur = node.Children[board[row][col] - 'a'];
            if (cur == null)
            {
                // 未找到匹配的路径
                return;
            }

            if(cur.word.Length > 0) { 
                // 这条路径匹配到一个单词
                // 单词添加到答案中
                ans.Add(cur.word);
                // 注意不需要 return, 还可以向后查找例如 app, apple
            }

            // 2. 设置本层的状态
            Visited[row*ColNum + col] = true;

            // 3. 递归下一层
            for(int i = 0; i < dRow.Length; i++)
            {
                // 新的状态
                int newRow = dRow[i] + row;
                int newCol = dCol[i] + col;

                if(newRow >= 0 && newRow < RowNum && newCol >= 0 && newCol < ColNum && !Visited[newRow * ColNum + newCol])
                {
                    DFS(board, cur, newRow, newCol);
                }
            }

            // 4. Status Reset
            Visited[row * ColNum + col] = false;

        }
    }
```  


