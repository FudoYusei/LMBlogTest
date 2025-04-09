---
layout: post
title: Algorithm (三) 链表
subtitle: 链表是非常常见的数据结构
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
priority: 3
---

![banner](/assets/images/Algorithm/Algorithm_Overview.png)  

## 链表
链表和数组是最常见的基础数据结构. 数组拥有索引随机存取的特性, 对于查找友好 O(1), 增删查不友好 O(n).  
而链表的优缺点与数组相反:  
1. 对于增删友好 O(1)
2. 对于查找不友好 O(n)

链表在算法中的用法, 多用于需要经常增删的结构:  
1. 树结构(多叉树基本只能用链表)
2. 前缀树
3. 优先队列

## LRUCache
[LRUCache](https://leetcode.cn/problems/lru-cache/submissions/545388139/)  

Least Recent Cache 最近缓存机制: 
1. 最近使用的缓存优先级最高  
2. 当缓存满了之后, 删除最久未使用的缓存元素

### HashMap
缓存的主要功能就是:  
1. 查找命中, 返回缓存的值, 未命中返回-1
2. 插入缓存
3. 删除缓存
直接使用 HashMap 这种基础数据结构就可以, HashMap 自带的 API 就可以满足  

### 双向链表
<span style='color:#4cd137'>链表的查找效率低, 但是可以额外使用一个 HashMap 解决这个问题, 键值对为 {key, ListNode}</span>  
HashMap 虽然能解决查找问题, 但是无法解决容量与优先级删除问题. 而链表可以将优先级最高的放在首位, 优先级最低的放在末尾, 相当于一个优先队列.   

因此需要一个双向链表来完成优先级队列的功能:  
1. 头部插入链表节点
2. 尾部删除链表节点
3. 移动节点到头部

```csharp
   public class LRUCacheTest2
   {
       int capacity;
       int count;

       DLinkList linkList
           ;
       public Dictionary<int, DLinkListNode> cache;

       public LRUCacheTest2(int capacity)
       {
           count = 0;
           linkList = new DLinkList();
           cache = new Dictionary<int, DLinkListNode>();
           this.capacity = capacity;
       }

       /// <summary>
       /// 获取缓存
       /// </summary>
       /// <param name="key"></param>
       /// <returns></returns>
       public int Get(int key)
       {
           if (cache.ContainsKey(key))
           {
               var node = cache[key];
               linkList.MoveToHead(node);
               return node.Value;
           }
           else
           {
               return -1;
           }
       }

       /// <summary>
       /// 插入缓存
       /// </summary>
       /// <param name="key"></param>
       /// <param name="value"></param>
       public void Put(int key, int value)
       {
           DLinkListNode node;
           // 新插入的缓存一定是位于队列头部
           if(cache.ContainsKey(key))
           {
               node = cache[key];
           }
           else
           {
               node = new DLinkListNode();
               count++;
               if(count > capacity) {
                   // 移除最久未使用的节点
                   linkList.RemoveTail();
                   // 记得从缓存中删除
                   cache.Remove(key);
               }
           }
           node.Key = key;
           node.Value = value;
           // 记得添加进缓存
           cache[key] = node;

           linkList.MoveToHead(node);
       }

       public class DLinkListNode
       {
           public int Key;
           public int Value;

           public DLinkListNode Next;
           public DLinkListNode Prev;
       }

       public class DLinkList
       {
           private DLinkListNode dummyHead;
           private DLinkListNode dummyTail;


           public DLinkList()
           {
               dummyHead = new DLinkListNode();
               dummyTail = new DLinkListNode();

               dummyHead.Next = dummyTail;
               dummyTail.Prev = dummyHead;
           }

           public void MoveToHead(DLinkListNode node)
           {
               // 先从队列中移除
               DLinkListNode removed = RemoveNode(node);
               // 插入到头部
               Insert(node);
           }

           /// <summary>
           /// 插入头部节点
           /// </summary>
           /// <param name="node"></param>
           public void Insert(DLinkListNode node)
           {
               DLinkListNode next = dummyHead.Next;

               dummyHead.Next = node;
               node.Prev = dummyHead;

               node.Next = next;
               next.Prev = node;
           }

           /// <summary>
           /// 删除尾部节点
           /// </summary>
           public void RemoveTail()
           {
               if(Empty())
               {
                   return;
               }

               RemoveNode(dummyTail.Prev);
           }

           public bool Empty()
           {
               return dummyHead.Next == dummyTail;
           }


           public DLinkListNode RemoveNode(DLinkListNode node)
           {
               DLinkListNode prev = node.Prev;
               DLinkListNode next = node.Next;
               prev.Next = next;
               next.Prev = prev;

               return node;
           }
       }
   }
```  


## Trie 树
[前缀树](https://leetcode.cn/problems/implement-trie-prefix-tree/description/) Trie 树本质上是一个多叉树, 可以使用链表数据结构实现.  

### 前缀树节点
前缀树对于英文十分友好, 因为英文单词的路径是唯一的, 只要前缀树中有这个单词, 一定有唯一一条路径抵达.  

前缀树节点:  
1. Word 如果为空, 说明当前节点是前缀, 否则就是对应的单词
2. IsWord 标记, 如果只要求查找是否存在, 可以直接用标记简化处理
3. Children 子节点, 对应 26 个英文字母(数组处理起来方便)

```csharp
        public class TrieNode
        {
            public bool IsWord = false;
            public TrieNode[] Children;
            public char Ch;

            public TrieNode()
            {
                Children = new TrieNode[26];
            }

            public TrieNode(char ch, bool isWord = false) {
                Children = new TrieNode[26];
                Ch = ch;
                IsWord = isWord;
            }
        }
```  

### 前缀树实现
前缀树主要功能:  
1. 查询单词
2. 添加单词

前缀树的删除有点麻烦, 但题目并未要求.  

```csharp
   public class Trie
    {
        public class TrieNode
        {
            public string Word;
            public TrieNode[] Children;

            public TrieNode()
            {
                // 26 个英文字母
                Children = new TrieNode[26];
                Word = string.Empty;
            }
        }

        TrieNode dummyRoot;

        public Trie()
        {
            dummyRoot = new TrieNode();
        }

        /// <summary>
        /// 向前缀树中添加单词
        /// </summary>
        /// <param name="word"></param>
        public void Insert(string word)
        {
            TrieNode cur = dummyRoot;
            foreach(char c in word)
            {
                if (cur.Children[c-'a'] == null)
                {
                    // 创建新的子节点
                    cur.Children[c - 'a'] = new TrieNode();
                }

                // 注意放在条件判断外部
                cur = cur.Children[c-'a'];

            }

            // 记得为最终的节点添加单词标志
            cur.Word = word;
        }

        /// <summary>
        /// 搜索前缀的路径终点
        /// </summary>
        /// <param name="prefix"></param>
        public TrieNode SearchWithPrefix(string prefix)
        {
            TrieNode cur = dummyRoot;
            foreach(char c in prefix)
            {
                if (cur.Children[c-'a'] == null)
                {
                    return null;
                }
                else
                {
                    cur = cur.Children[c-'a'];
                }
            }

            return cur;
        }

        /// <summary>
        /// 从前缀树中搜索单词
        /// </summary>
        /// <param name="word"></param>
        /// <returns></returns>
        public bool Search(string word)
        {
            TrieNode prefix = SearchWithPrefix(word);
            return prefix != null && prefix.Word == word;
        }

        public bool StartsWith(string prefix)
        {
            return SearchWithPrefix(prefix) != null;
        }
```  
