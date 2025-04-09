---
layout: post
title: Algorithm (七) 动态规化
subtitle: 多样化的动态规化
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
priority: 7
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 动态规化
算法中最难的类型, 动态规化涉及到的题目多种多样, 从题目本身很难看出有什么规律. 需要进一步分析题目, 从题目描述的状态入手.  

动态规化的名称让人难以理解, 个人理解如下:  
1. 动态是核心, 我们需要从题目中提炼出状态以及状态方程, 也就是状态的变化.
2. 规化就是程序化, 通常来说就是递推

动态规化也可以称为动态递推.  
动态规化解题的一个小技巧就是, 因为人脑的函数栈非常有限, 所以题目涉及到递推的复杂度不会特别高, 因此寻找几个边界状态递推一下, 可以看出一点规律.  

## 解题步骤
1. 定义子问题, 分治  
2. 猜测递归方程 
3. 合并子问题
4. 递归 & 备忘录递归 & dp表从下往上递推
5. 解决原始问题  

> 即使知道步骤, 实际上也没什么用

## Fibonacci 数列
Fibonacci 数列是递归和动态规化的入门题目.   
这个题目中的状态参数有:   
1. 天数 i
2. 第 i 天兔子的数量 

题目求第 n 天时, 兔子的数量, 那么很明显 i 是自变量, 兔子数量是因变量 dp[i], 求 dp[n]  

```csharp
int Fib(int n)
{
    if(n == 0) return 0;
    if(n == 1) return 1;
    if(n == 2) return 2;

    int[] dp = new int[n];
    for(int i = 3; i < n; i++)
    {
        dp[i] = dp[i-1] + dp[i-2];
    }
}
```  

## 不同路径
[不同路径](https://leetcode.cn/problems/unique-paths/description/)  

动态规化的解题方法第一眼看上去都是不明觉厉, 状态方程是如此简洁优美, 彷佛应该存在一个状态方程是用来解这道题一样. 实际上是因为, 我们跳过了基础的状态分析阶段.  

动态规化也可以说是另一种形式的深度优先遍历加上备忘录的算法.  

### 暴力解法
DFS 遍历所有路径, 备忘录记录到达每个格子的不同路径总数便于重复使用, 不使用备忘录会超时.  

```csharp
public class Solution {

    int[] dx = [0, 1];
    int[] dy = [1, 0];

    public int UniquePaths(int m, int n) {
        memo = new int[m, n];
        for(int i = 0; i <m;i++)
        {
            for(int j = 0; j < n; j++)
            {
                memo[i, j] = -1;
            }
        }

        return DFS(m, n, 0, 0);
    }

    int[,] memo;
    public int DFS(int m, int n, int row, int col)
    {
        if(row == m-1 && col == n-1)
        {
            // 1. 递归返回条件
            return 1;
        }

        if(memo[row, col] != -1)
        {
            return memo[row, col];
        }

        int res = 0;
        for(int i = 0; i < dx.Length; i++)
        {
            int newRow = row + dx[i];
            int newCol = col + dy[i];

            if(newRow >= 0 && newRow < m && newCol >= 0 && newCol < n)
            {
                res += DFS(m, n, newRow, newCol);
            }
        }
        memo[row, col] = res;
        return res;
    }
```  

### 状态方程
通过分析答案来逆推状态方程的含义, <span style='color:#4cd137'>状态参数要足以覆盖所有状态以及能够确定唯一的状态.</span> 状态方程 dp[状态参数] 求出的值也是唯一的, 本题要求到达网格坐标 [m, n] 处的路径总数, 因此可以明确得到状态方程 dp[m][n] = 到达[m, n] 处的不同路径总数.  

二维网格是动态规化常客, 因为二维网格具有以下共性:  
1. 二维网格的每一个格子都可以方便的使用其索引代表唯一一个确定的状态 [row, col]
2. 二维网格格子间的移动路径可以视为一个确定的状态转移. 
   1. 例如本题, 只能向下或者向右移动, 因此可以得出一个简单的状态方程 dp[row][col] = dp[row-1]dp[col] + dp[row][col-1]

### 最优子问题
现在先假定 dp[m][n] 代表从索引 [0][0] 处到达索引 [m][n] 的不同路径. <span style='color:#fbc531'>这时候会法线一个矛盾点, 即两点之间路径是不唯一的.</span> 但是前面又说了 dp[m][n] 是唯一一个确定的值.  

这里就是动态规化的另一个特性: 最优子问题. 我们可以得到到达 [m][n] 的所有不同路径, 但是我们只需要一个局部最优解, 因此动态规化的问题都会有聚合状态的特性, 例如 最短路径 最长路径 不同路径数  

### 无后效性
<span style='color:#fbc531'>为什么这一题要限定机器人只能向下或者向右移动?</span>  
有一个很重要的点就是, 只有这样才能保证后效性, 即状态推导只需要基于当前状态. 有无后效性的区别详见不同路径III  


### 动态规化
1. 从状态推移得到动态方程: dp[m][n] = dp[m-1][n] + dp[m][n-1]  
2. 确定初始值(边界值): 边缘行和列的值都是可以直接确定的

```csharp
public class Solution {
    int[] dx = [0, 1];
    int[] dy = [1, 0];

    public int UniquePaths(int m, int n) {
        int[,] dp = new int[m, n];
        for(int i = 0; i < m; i++)
        {
            dp[i,0] = 1;
        }

        for(int j = 0; j < n;j++)
        {
            dp[0,j] = 1;
        }

        for(int i = 1;i < m; i++)
        {
            for(int j = 1; j < n; j++)
            {
                for(int k = 0; k < dx.Length;k++)
                {
                    dp[i,j] += dp[i-dx[k], j-dy[k]];
                }
            }
        }

        return dp[m-1, n-1];
    }
}
```  
可以看到动态规化的代码更加精炼, 因为动态规化的复杂度都集中在状态方程的分析上. 动态规化算法题的难点就在于题目多种多样, 状态参数也是多种多样.  

> 动态规化的本质就是算法的本质, 不同于我们之前做题学习算法和数据结构, 首先告诉你一种算法或者数据结构的特性, 然后去套用在解题方法上.
> 动态规化的解题思路才是现实问题的解法, 定义问题, 找到问题的所有解法, 从所有解法中总结出公式
> 要证明公式正确, 那这个公式必然能覆盖所有解决方法  


## 不同路径III
这道题的最大区别是, 机器人可以上下左右移动, 但不能重复经过同一个格子, 并且最终要经过所有能移动的格子.     
1. 对于 1x1 的格子, 走到右下角不同路径为 1 -> dp[0][0] = 1
2. 对于 2x2 的格子, 走到右下角不同路径为 2 -> dp[1][1] = 2
3. 对于 3x3 的格子, dp[0][0] = 1 dp[1][1] = 6

因为 dp 值不仅仅与当前状态相关, 它可能要基于后序的状态: dp[m][n] = dp[m-1][n] + dp[m][n-1] + dp[m+1][n] + dp[m][n+1]  
这样显然与动态规化从当前状态向下一个状态转移不符. 造成这种无法确定唯一状态的原因多半是状态参数缺失.  
个人理解, 有一种更加简单的判别方法, 我们在程序中的状态推移是有顺序的. 在不同路径算法中  

### 暴力解法 
实际上使用深度优先加上备忘录解题的时候就会发现不对劲的地方. 已经计算出来的备忘录的值会在后面的递归过程中被更新.  

### 状态方程
<span style='color:#4cd137'>既然 m 和 n 无法确定状态, 再添加一个记录经过的格子的状态参数</span> 并且题目要求到达终点时, 需要经过所有能移动的格子.  

状态参数:  
1. status: 使用 Bitmap 记录 m x n 个格子的经过状态, 题目给出限制条件 m x n <= 20, 因此一个 int 整数即可存储当前状态下已经经过的格子(最多使用 20 位). 
2. 当前格子索引: 将二维索引转换成一维索引  

现在状态参数完全足够定义一个唯一的状态. 已经计算过的状态不会再次被计算. 说明在递推过程中, 所有的状态都是唯一的.  

> 这种定义方法没见过, 基本也想不出来, 只能根据题意得到, 脱离现实的三维状态参数, 人脑也基本无法想象了. 关键点在于状态的推移

<span style='color:#4cd137'>递归方法传入的参数就是该层的状态, 等同于在函数内部设置.</span>  

```csharp
public class Solution
{
    int[] dRow = [0, 0, 1, -1];
    int[] dCol = [1, -1, 0, 0];

    int m_RowNum;
    int m_ColNum;

    Dictionary<int, int> memo;

    public int UniquePathsIII(int[][] grid)
    {
        m_rowNum = grid.Length;
        m_ColNum = grid[0].Length;

        int status = 0;
        int startRow = 0;
        int startCol = 0;
        memo = new Dictionary<int, int>();

        // 1. 设置初始状态的值 status 的位 0 - 表示无法经过的格子 1 - 表示可以经过的格子
        for(int row = 0; row < rowNum; row++)
        {
            for(int col = 0; col < colNum; col++)
            {
                if(grid[row][col] == 1)
                {
                    // 1 - 起始格
                    startRow = row;
                    startCol = col;
                }
                else if(grid[row][col] == 2 || grid[row][col] == 0)
                {
                    // 2 - 结束方格 0 - 空方格
                    status |= (1 << (row * colNum + col));
                }
                else if(grid[row][col] == -1)
                {
                    // -1 - 障碍物 默认为0
                }
            }
        }

        // status 对应位置的bit : 1- 

        return DFS(grid, startRow, startCol, status);
    }

    public int DFS(int[][] grid, int row, int col, int status)
    {
        // 1. 递归返回条件
        if(grid[row][col] == 2)
        {
            if(status == 0)
            {
                // 到达结束方格, 并且经过了所有的可行走格子
                return 1;
            }else{
                // 到达结束方格, 但没有经过所有可移动的格子
                return 0;
            }

        }

        // 2. 备忘录 index 
        // ((row*colNum) + col) << (rowNum * colNum) + status 也可以, 高位和低位互相不影响
        // 高位是
        int index = ((row*colNum) + col) << (rowNum * colNum) | status;

        if(memo.ContainsKey(index))
        {
            return memo[index];
        }

        // 2. 因为这次方法没有局部变量, 都是直接使用参数传递状态, 因此不需要计算当前层的状态

        // 3. 递归下降, 计算下一层的状态, 传参给下一层
        int res = 0;
        // 3.1 从 status 中计算得出能够移动到的下一层
        for(int i = 0; i < dRow.Length; i++)
        {
            int newRow = row + dRow[i];
            int newCol = col + dCol[i];

            // 剔除掉不能移动的格子
            if(newRow < 0 && newRow >= m_RowNum && newCol < 0 && newCol >= m_ColNum)
            {
                continue;
            }

            // 判断 newRow 和 newCol 是否能够移动,  
            if(status & (1 << (row * colNum + col)) == 0)
            {
                // 对应位置 bit 为 0 表示无法移动
                continue;
            }

            // 对应位置 bit 置为 0 表示该格子已经经过
            // 传参省去了重置状态的麻烦
            int newStatus = status ^ (1 << (row * colNum + col));
            res += DFS(grid, newRow, newCol, newStatus);
        }

        // 保存到备忘录中
        memo.Add(index, res);

        // 4. 所有局部状态使用函数传参代替, 所以不需要重置局部状态
        return res;
    }
}
```  

## N皇后
[N皇后](https://leetcode.cn/problems/n-queens/description/)是一个经典的动态规化难题, 光看题目就能把人看晕, 无从下手.  

### 状态操作
无从下手的题目, 一般是状态参数难以辨别, N 皇后问题中的状态操作很简单, 就是在棋盘上落下棋子. 落下棋子后的状态变化:  
1. 同一行的其他位置无法再次落子
2. 同一列的其他位置无法再次落子
3. 该棋子所处的两条对角线无法再次落子

同时, 因为每一行只能落一个子, 每一行递进落子作为自变量:  
1. 当前操作: 在 row 行落子 A, 找到 row 行能够落子的列, 选择其中一列作为落子点
2. 当自递进到 row + 1 行时, 当前状态为:
   1. 棋子 A 所处的 col 列无法落子
   2. 棋子 A 所处的两个对角线无法落子


## 鸡蛋掉落
[鸡蛋掉落](https://leetcode.cn/problems/super-egg-drop/solutions/28523/887-by-ikaruga/)  

也是一个典型的题目都看不懂的动态规化问题.  

换一种角度来理解题目, 最小操作数 N 可以可以覆盖所有楼层. 进行 N 次鸡蛋操作, 可以最多覆盖的层数  

### 状态参数
1. 鸡蛋数量 m  
2. 楼层层数 n  

题目要求操作数, 那么 dp[m, n] 为 m 个鸡蛋测出 n 层楼的最小操作数  

假设在第 x 层进行一次鸡蛋操作, 那么状态转移有两种情况:  
1. 鸡蛋碎掉, 排除掉 x 以及 x 往上的层数, 状态转移 dp[m, x-1]
2. 鸡蛋没碎, 排除掉比 x 低的层数, 状态转移 dp[m-1, n-x] 
与二分查找法一模一样. 状态转移被分解为互斥的两个子状态, 考虑到要完全覆盖两个子状态, 所以操作数需要取两个子状态中最大的.  

在 x 楼层扔下一个鸡蛋的状态函数: dp[m, n] = 1 + max(dp[m, x-1], dp[m-1, n-x])  
因为 x 值不固定, 会导致需要的操作数不同, 所以题目要求的是最少的操作数  
$$ dp(m,n) = 1 + \min_{1\leq x\leq n}(\max(dp(m-1, x-1), dp(m, n-x))) $$

初始值 1 个鸡蛋 n 层楼 dp[1, x] = x  
初始值 m 个鸡蛋 1 层楼 dp[m, 1] = 1  

```csharp
public int SuperEgg(int k, int n)
{
    return DFS(k, n);
}

int m_Level = 100;
public int DFS(int k, int n)
{
    if(k == 0 || n == 0)
    {
        return 0;
    }
    if(k == 1)
    {
        // 1 个鸡蛋
        return n
    }

    if(n == 1)
    {
        // 1 层楼
        return 1;
    }

    int res = 0;
    for(int x = 1; x <= m_Level; x++)
    {
        // 在 x 楼层鸡蛋掉落, 操作数 + 1
        // 没碎 -> DFS(k, x-1)
        // 碎了 -> DFS(k-1, n-x)
        res = Math.Min(res, 1 + Math.Max(DFS(k, x-1), DFS(k-1, n-x)));
    }

    return res;
}
```  

### 逆思维
<span style='color:#4cd137'>鸡蛋操作实际上有一个隐含的逻辑, 鸡蛋碎掉排除掉更高的楼层, 鸡蛋没碎排除掉更低的楼层. 而剩余楼层的操作并不会受到高度影响. 也就是说 [1,10] 和 [91,100] 这十层楼对于鸡蛋操作是一样的. 动态规化典型的最优子问题特性</span>  

先从边界状态开始推算有助于理解状态转移和操作:   
1. 1 个鸡蛋, T 次操作, 最多覆盖 T + 1 层楼, 因为无法保证鸡蛋是否会碎, 需要从最底层往上扔
2. K 个鸡蛋, 1 次操作, 最多覆盖 2 层楼, 因为只能操作一次, 所以还是最底层扔一次, 最底层碎了, 确定最底层-1, 没碎只能确定最底层

假设有 m 个鸡蛋, n 次操作. 这里的操作有点像数字猜大小, 每次操作有两种走向:   
1. 鸡蛋碎掉, 状态转移 dp[m-1, n-1]
2. 鸡蛋没碎, 状态转移 dp[m, n-1]
注意, 因为 dp 值代表的是覆盖范围, 所以 dp[m,n] = dp[m-1, n-1] + dp[m, n-1]  

```csharp
public class Solution {
    public int SuperEggDrop(int k, int n) {
        int t = 1;

        while(CalcF(k, t) < n+1)
        {
            t++;
        }

        // 此时t对应的值是大于等于n+1
        return t;
    }

    int CalcF(int k, int t)
    {
        if(k == 1 || t == 1)
        {
            return t+1;
        }

        return CalcF(k-1, t-1) + CalcF(k, t-1);
    }
}
```  


