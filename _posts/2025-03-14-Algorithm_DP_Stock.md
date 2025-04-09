---
layout: post
title: Algorithm (八) 动态规化-买卖股票
subtitle: 经典的 DP 入门系列题
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
priority: 5
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 动态规化
买卖股票是动态规化的系列题, 题目很好的展现了动态规化从简到难的本质: 状态参数增多, 状态转移步骤变复杂. <span style='color:#4cd137'>本质上, 还是找到能够确定唯一状态并且覆盖所有状态的参数以及状态转移方程.</span>  

1. 通常最容易分析出来的是状态转移操作
2. 

## 买卖股票
[买卖股票](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/description/)  

经典的动态规化基础题目, <span style='color:#4cd137'>动态规化并不是银弹, 不如说先有了解决方法才能写出对应的动态规化程序</span>  

第 n 天的收益只与 n-1 天的收益加上今天的买卖相关, 只要明白这一点, 就很容易得出状态方程的两个状态参数  
1. 天数 n, 属于自变量, 类似于只能向右走的机器人 
2. 持有股票的状态 0-代表为持有股票 1-代表持有股票

状态转移操作:  
1. 天数是自变量
2. 股票持有状态由前一天状态和当天操作共同决定

```csharp
public class Solution {
    public int MaxProfit(int[] prices) {
        int n = prices.Length;
        if(n == 0)
        {
            return 0;
        }
        if(n == 1)
        {
            return 0;
        }
        int[,] dp = new dp[n+1];
        dp[0,0] = 0;
        dp[1,0] = 0;
        dp[1,1] = -prices[0];

        for(int i = 1; i <=n; i++)
        {
            dp[i, 0] = Math.Max(dp[i-1,0], dp[i-1,1]+prices[i-1]);
            dp[i, 1] = Math.Max(dp[i-1,1], dp[i-1,0] - prices[i-1]);
        }

        // 无论如何, 肯定卖掉手中的股票收益更高. Max(dp[n,0], dp[n,1])
        return dp[n, 0];
    }
}
```  

## 买卖股票IV
[买卖股票III](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iii/)的难度不断增加, 本质上状态参数增加和状态转移的复杂性增加.  
本题添加了一个新的限制条件, 最多完成两笔交易(买卖), 实际上就是增加了一个新的状态参数:  
1. 天数 n, 自变量
2. 持有股票状态
3. 完成交易次数

对于复杂的状态, 可以使用归纳法找规律:   
1. 交易次数为 0, 持有股票状态为 0 时, 所有的 dp[n, 0, 0] 值都为 0
2. 交易次数为 0, 持有股票状态为 1 时, dp[n, 1, 0] 的值为 Math.Max(dp[n-1, 1, 0], dp[n-1, 0, 0] - prices[n])

同时, 存在一些不可能存在的状态, 这些状态需要赋予特殊值(最大值或者最小值), 取决于这些状态不会影响到状态转移过程, 例如, dp[0, 0, 1] 第一天不可能完成一次交易. 所以该状态不可能存在, 又因为题目答案是求最大值, 所以将这些不可能存在的状态全部赋予最小极限值, 这样它们就不会影响正常的状态推移.  

<span style='color:#4cd137'>股票问题有一个共同的逻辑, 引起状态转移的操作有: </span>  
1. 不操作
2. 买股票
3. 卖股票  
和二维网格中的走格子的方向一样.  

[买卖股票IV](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iv/description/) 的解可以覆盖买卖股票 III  

```csharp
public class Solution{
    public int MaxProfit(int k, int[] prices)
    {
        int n = prices.Length;
        if(n == 0)
        {
            return 0;
        }
        if(n == 1)
        {
            return 0;
        }
        
        int[,,] dp = new int[n+1, 2, 3];
        // 默认值设置
        for(int i = 0; i <= n; i++)
        {
            for(int j = 0; j <= 1; j++)
            {
                for(int k1 = 0; k1 <= k;k1++)
                {
                    // 不能使用最小极限值, 会溢出
                    // dp[i,j,k] = int.MinValue;
                    dp[i, j, k1] = -10000000;
                }
            }
        }
        // 1. 设置初始状态的值
        // 第一天的 dp 值
        dp[1, 0, 0] = 0;
        dp[1, 1, 0] = -prices[0];
        for(int i = 1; i <= n; i++)
        {
            // 未进行过交易
            dp[i, 0, 0] = 0;
            // 只执行过第一次买入交易
            dp[i, 1, 0] = Math.Max(dp[i-1, 1, 0], dp[i-1, 0, 0] - prices[i-1]);
        }

        

        // 2. 状态转移
        for(int i = 2; i <= n; i++)
        {
            // dp[i, 0, 0] 0 持有, 0 交易次数 
            // dp[i, 1, 0] 1 持有, 0 交易次数 买股票

            // dp[i, 0, 1] 0 持有, 1 交易次数. 卖出股票
            // dp[i, 0, 1] = Math.Max(dp[i - 1,0, 1], dp[i - 1, 1, 0] + prices[i-1]);


            // dp[i, 1, 0] 1 持有, 1 交易次数. 买入股票
            // dp[i, 1, 1] = Math.Max(dp[i - 1, 1, 1], dp[i - 1, 0, 1] - prices[i-1])


            // dp[i, 0, 2] 0 持有, 2 交易次数. 卖出股票
            // dp[i, 0, 2] = Math.Max(dp[i - 1, 0, 2], dp[i - 1, 1, 1] + prices[i-1])

            // dp[i, 1, 2] 1 持有, 2交易, 买入股票, 对于买卖股票 III, 两次交易后不会再次购买

            // 合并买入卖出操作为通解
            for(int j = 1; j <= k; j++)
            {
                // 卖出操作
                dp[i, 0, k] = Math.Max(dp[i-1, 0, k], dp[i-1, 1, k-1] + prices[i-1]);
                // 买入操作
                dp[i, 1, k] = Math.Max(dp[i-1, 1, k], dp[i-1, 0, k] - prices[i-1]);
            }
        }

        // // 两次交易完成
        // return Math.Max(dp[n, 0, 0], Math.Max(dp[n,0,1], dp[n, 0, 2]));
        
        // k 次交易完成
        int res = 0;
        for(int i = 1; i  <= k; i++)
        {
            res = Math.Max(res, dp[n, 0, i]);
        }
        return res;
    }
}
```  

## 买卖股票IV

