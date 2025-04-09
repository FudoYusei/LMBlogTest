---
layout: post
title: Algorithm (八) 树
subtitle: 树结构适用于分治算法
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
priority: 8
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

树和图都是一种逻辑结构, 树的每个节点都拥有父子结构. 因为树的父子结构, 经常用于在动态规化中的思路中解决最优子结构.    

树的分支结构也适用于算法的条件选择分支, 例如决策树.  

## 数据结构
```c
class TreeNode
{
    int val;
    TreeNode* left;
    TreeNode* right;

    TreeNode(int x) : val(x), left(NULL), right(NULL){}
};
```  

前序遍历 中序遍历 后序遍历  

## 二叉搜索树
有序二叉树, 左子树一定全部小于父节点, 右子树一定全部大于父节点  

有序二叉树的中序遍历结果就是一个有序的数列.  

```c
goid inorder(TreeNode* node)
{
    if(node == nullptr)
    {
        return;
    }

    inorder(node->left);
    dosomething(node);
    inorder(node->right);
}
```  

二叉树的定义就是, 节点的左子树一定全都小于给节点, 右子树一定全部大于该节点  
这种有点像二分查找.  
   A
      B  
     C   
C作为A节点右子树的左子树也是大于A的  

这也是为什么有序二叉树的中序遍历也是有序的  
<font color=#4cd137>左子树的所有节点小于当前节点, 右子树的所有节点大于当前节点. 并不仅仅是左儿子节点或者右儿子节点</font>  

二叉搜索树的增删改查都是O(logn), 也就是平均来说都是树的高度  
除非二叉搜索树退化成链表. 因此会有自平衡二叉搜索树  

## 为什么和树相关的算法多是递归  
树本身的结构不适合于数组的遍历  

## 树中序遍历 inorder traversal
https://leetcode.cn/problems/binary-tree-inorder-traversal/description/  

### 中序遍历一个特殊的方法 栈
递归实际上就是函数自动压栈, 将我们的数据自动压入了编译系统开拓的栈中.  
<font color=#4cd137>调用函数时, 调用者的数据就是压栈, 函数参数就是下一轮操作的数据. 退出函数就是出栈</font>  

中序遍历递归形式  
```c
void inorder(TreeNode* node)
{
    if(node == nullptr)
    {
        return;
    }

    inorder(node->left);
    travel.push_back(node.val);
    inorder(node->right);
}
```  

中序遍历栈形式  
栈具有先进后出的特性.  
函数调用本质上就是调用者的数据压栈出栈  

递归调用的函数压栈  
调用时压栈, 退出时出栈  
对于递归并没有什么左右孩子, 只有先后调用顺序  
递归就是先进后出, 后进先出, 每次递归函数调用就是将调用者还需要使用的局部变量压栈   

<font color=#fbc531>树的递归需要压栈的局部变量是什么?</font>  

<font color=#fbc531>树的先序中序后序遍历时, 树的访问路径有区别吗?</font>  

递归左右孩子树的问题在于.  
递归左孩子确实可以使用循环, 一致遍历到左孩子为空.  
但是递归右孩子时, 并不能使用递归, 一直遍历到右孩子为空.  
因为递归方法并不会去认识右孩子, 而是同样先遍历左孩子, 然后才是右孩子  

因此递归左右子树的递归函数实际上要完整实现  
栈的作用就是保存函数的状态, 直到真正执行的时候从栈中pop出来, 进行处理  

<font color=#4cd137>递归调用转换为栈的难点之一在于, 函数调用压入栈的不止局部数据, 还有返回地址. 因此我们压入栈的除了节点, 还需要一个状态值, 表示接下来该执行右孩子递归</font>  


```c
vecotr<int> inorder(TreeNode* root)
{
    vector<int> res;

    stack<TreeNode*> stk;
    while(root!=nullptr || !stk.empty())
    {
        while(root != nullptr)
        {
            stk.push(root);
            root=root->left;
        }

        // 上一层出栈
        root= stk.top();
        stk.pop();

        // 函数执行
        res.push_back(root->val);
        root = root->right;
    }

    return res;
}
    
```  

stack中存储两个值, 一个是节点的值, 一个是状态值表示这个节点是左孩子还是右孩子  

函数调用在压栈的是什么?  

栈底  
...  

调用者的栈帧(局部变量) 通常使用ebp寄存器指向栈帧的底部  


栈顶 通常esp寄存器指向栈顶  
<font color=#4cd137>每次函数调用压栈, 压入的是函数调用时的参数, 并不是调用者的局部变量. 调用者的局部变量实际上就是调用者的栈帧, 对应的是压入的旧的ebp寄存器中保存的值 </font>  

栈顶的内容通常是这三个:  

参数  
旧的ebp值  
返回地址   

旧的ebp值, 实际上是为了找到上一层栈帧的位置. 因为每个调用者本身局部数据大小不一致, 在栈中地址不一样, 需要主动保存  
在递归调用中, 每一层的数据都是固定的, 和参数也是一致的, 只需要入栈出栈一次即可, 因此不需要主动保存旧的ebp值  
返回地址是需要的, 因为整个递归中只有三个步骤, 因此使用状态值表示函数执行顺序  
参数值, 只需要在递归调用的时候压入对应的值即可  

现在压入的是参数和返回状态.  
注意返回状态不是和参数一起使用的.  
返回状态是在参数使用完毕后, 返回上一层时使用的. 告知上一层接下来从哪里开始执行  

例如, 无论递归调用时传入的参数是node->left还是node->right, 函数都会执行完整的流程.  
返回状态只会影响函数调用完毕时从哪里开始执行. 返回状态针对的是上一层的参数. 因此实际上有两个状态, 一个是当前层的状态, 一个是上一层的状态  
函数开头总是要得到当前层的参数, 也就是stk.top(), 当然不需要pop(), 而是接着往下执行.  
先得到当前层的状态 curStatus=prevStatus;   
同时保存栈中得到的上一层状态 prevStatus = stk.top().second;  

根据当前层状态决定接下来的执行操作.  

这个递归调用总共就两个返回点.  
一个是接着递归调用右孩子, 一个是直接返回  

递归调用右孩子时, 保存的返回状态是2  

<font color=#4cd137>函数调用本身除了压栈, 还存在跳转</font>  
因此我们需要在stk.push之后, 接着跳转到函数开头  

函数调用压栈, 跳转到函数地址. 也就是Continue的时候, 不需要返回状态, 或者或应该设置跳转地址为函数开头  

函数执行完毕, 出栈, 获取返回地址. 这里是获取返回状态, 返回状态实际上也是设置跳转地址  

因此本质上函数调用和函数返回都是设置跳转地址, 而函数中一共有三个地方需要跳转  

函数开头 goto=0  
第一次递归调用 goto=1  
第二次递归调用 goto=2  

对应需要跳转地址进行设置的地方  
递归调用函数的地方  
函数结束的地方  

函数执行顺序  
1. 从栈中得到函数参数, 复制到自己的栈帧中. 我们直接节省掉参数复制. 因此只需要直接获取栈顶元素即可  
2. 判断当前节点是否为空, 为空则函数返回到上一层, 也就是出栈当前栈帧, 得到上一层的返回地址, 我们使用返回状态代替
3. 如果返回状态为1, 说明是第一个递归调用返回. 如果为2, 则第二个调用返回.

```c
vecotr<int> inorder(TreeNode* root)
{
    vector<int> res;
    stack<pair<TreeNode*, int>> stk;

    // 0 - 未执行 1 - 左孩子调用已执行, 相当于返回地址为
    stk.push(pair(root, 0));
    int gotoId = 0;
    
    while(!stk.empty())
    {
        // 栈顶正好是当前的参数
        auto cur = stk.top();
        TreeNode* curNode = cur.first;
       
        if(gotoId == 0)
        {
            // 函数开头
            if(curNode == nullptr)
            {
                // 返回上一层, 函数返回
                // 函数返回, 首先出栈自己的局部数据  
                auto ret = stk.top();
                stk.pop();
                // 然后设置跳转地址为栈中获取的返回地址, 返回地址在函数调用时同样入栈
                gotoId = ret.second;
            }

            // 第一次递归调用
            // 正常函数调用, 参数压栈,设置返回地址为第一次递归调用完毕
            stk.push(pair(curNode->left, 1));
            // 正常函数调用, 设置跳转地址
            gotoId = 0;
            continue;
        }

        if(gotoId == 1)
        {
            // 第一次递归调用的返回点

            res.push_back(curNode->val);

            // 第二次递归调用
            // 同样压栈参数+返回地址
            stk.push(pair(curNode->right, 2));
            // 设置跳转地址为函数地址
            gotoId = 0;
            continue;
        }

        if(gotoId == 2)
        {
            // 第二次递归调用的返回点
            // 没有任何操作, 直接返回上一层
            // 出栈局部变量
            auto ret = stk.top();
            stk.pop();
            // 设置跳转地址为栈中获取的函数返回地址
            gotoId = ret.second;
            continue;
        }
    }
}
```  

<font color=#fbc531>为什么这个栈的方法不需要压入函数返回地址???</font>  

<font color=#4cd137>每一轮的出栈代表了什么?</font>  
函数的数据什么时候会出栈. 当然是函数执行完毕的时候会出栈.  

在这个递归函数中. 递归调用函数的地方有两个, 因此函数执行完毕的地方有两个  
<font color=#4cd137>入栈并不是执行函数</font>  
递归调用函数等同于入栈, 需要注意的点实际上是函数在哪里执行.  
当出栈的时候, 就是函数需要执行的时候.  

inorder的函数执行体前半段被转换成了 while循环  

inorder(root->left) 递归调用 => 当前节点自身入栈, 所有左孩子节点入栈  
直到达到递归退出条件, 在某个节点为null, 直接返回. 函数返回意味着返回调用者上层次, 也就是出栈 => root=stk.top(); stk.top();  
继续执行, 到了递归调用中的函数执行体 => res.push_back(root->val);  
再次遇到inorder(root-right)调用 => 递归调用的局部重复性, 只需要让参数入栈, 然后再次回到开头, 将节点压入栈, 再一次将其左孩子节点压栈.  

## 后序遍历
<font color=#4cd137>注意上面算法的最后一次调用并没有将当前节点压栈, 因为后续没有使用当前节点的局部数据了</font>  
对比后序遍历  
后序遍历的操作在两个递归调用之后, 需要  
  

<font color=#fbc531>如果有三个递归调用呢? 只有尾递归可以优化?</font>  
  

<font color=#fbc531>那么后序遍历呢?</font>  

## 树遍历的一种简单的栈解法  
在树的遍历的时候, 就将节点压入栈. 先序遍历  

```c
public static void preOrder(TreeNode* root)
{
    Stack<TreeNode*> stk;

    stk.push(root);

    while(!stk.empty())
    {
        TreeNode* node = stk.top();
        stk.pop();

        // 先序遍历
        visit(node);

        // 先入后出
        if(node->right != null)
        {
            stk.push(node->right);
        }

        if(node->left != null)
        {
            stk.push(node->left);
        }
    }
}
```  
虽然入栈的操作和递归调用不一样, 但是顺序是一样的, 是某种巧合  

这种思路就是模拟出正确的数据入栈出栈操作的顺序  



## 递归与栈
https://cloud.tencent.com/developer/article/1705922  
它的栈转换方法不一样.  
它是直接规定好了左右子节点的入栈顺序.  
而实际上的入栈时机并没有像它这么提前, 它的顺序是对的  

# 从前序遍历与中序遍历构造二叉树
https://leetcode.cn/problems/construct-binary-tree-from-preorder-and-inorder-traversal/solutions/255811/cong-qian-xu-yu-zhong-xu-bian-li-xu-lie-gou-zao-9/  

前序遍历特点:  
[根节点, [左子树的遍历结果], [右子树的遍历结果]]  

中序遍历特点:  
[[左子树遍历结果], 根节点, [右子树遍历结果]]  

从前序遍历中确定根节点. 前序遍历确定左右子树的根节点, 就需要得知左右子树的节点数量  
匹配中序遍历时先获取前序遍历给的根节点, 匹配根节点得到左右子树的节点数量  
左右子树的第一个节点就是树的根节点
递归  
1. Start End 代表在前序遍历的集合中一个树的开端和终端. 递归返回条件, start==End, 返回当前节点  
2. 因此, start索引处节点必然是这个树的根节点    
3. 在中序遍历序列中匹配根节点, 得到树的左右子树的开端和终端  
4. 当前节点的左孩子 = 递归调用左子树(leftStart, leftEnd)
5. 当前节点的右孩子 = 递归调用右子树(rightStart, rightEnd)
6. 返回当前节点

<font color=#4cd137>有了递归返回条件还需要返回当前节点?</font>  
什么时候除了递归结束条件, 还需要额外的返回值?  
<font color=#4cd137>递归的返回值本就是正常的, 并不是额外的返回值</font>  

# N叉树的先序遍历  
N叉树的先序 中序 后序遍历, 递归都是一个套路  
递归调用左子树和递归调用右子树实际上并没有任何区别:  
```c
for(int child : node->children)
{
    preorder(child);
}
```  

<font color=#4cd137>使用栈来模拟函数调用可以发现为什么编译器会将循环变成状态机, 因为方便</font>  

# 树的层序遍历

https://leetcode.cn/problems/n-ary-tree-level-order-traversal/  
注意这里的答案是每一层都单独分出来作为一个集合. 而不是整体一个集合, 因此需要一个变量记录一下每一层有多少个元素.  
```c
class Solution{
public:
    vector<vector<int>> levelOrder(TreeNode* root)
    {
        if(root == nullptr)
        {
            return {};
        }

        vector<vector<int>> ans;

        Queue<TreeNode*> q;
        q.push(root);

        while(!q.empty())
        {
            // 当前层的元素个数
            int cnt = q.size();
            vector<int> level;

            // 根据元素个数, 辨别在同一层的元素
            for(int i = 0; i < cnt; i++)
            {
                TreeNode* curNode = q.top();
                q.pop();
                level.push(curNode->val);

                for(TreeNode* child: curNode->children)
                {
                    q.push(child);
                }


            }
            ans.push(move(level));
        }

        return ans;
    }
};
```  
