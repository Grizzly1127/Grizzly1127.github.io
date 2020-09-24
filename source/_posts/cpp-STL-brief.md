---
title: C++ STL库底层数据结构
date: 2020-07-17 11:56:39
tags: 
- c++
- stl
categories: 
- c++
copyright: true
---

STL容器可以分为以下几大类：
① 序列容器：vector、list、deque、string
② 关联容器：set、multiset、unorderd_set、unorderd_multiset、map、multimap、unorderd_map、unorderd_multimap
③ 其他容器：stack、queue、valarray、bitset
<!-- more -->

## 1. vector

内部数据结构：**数组**。
vector 数组动态增加大小，并不是在原空间之后持续新空间（因为无法保证原空间之后尚有可供配置的空间），而是以原大小的两倍另外配置一块较大的空间，然后将原内容拷贝过来，然后才开始在原内容之后构造新元素，并释放原空间。因此，对 vector 的任何操作，一旦引起空间重新配置，同时指向原vector 的所有迭代器就都失效了。
vector 常用来保存需要经常进行随机访问的内容，并且不需要经常对中间元素进行添加删除操作。

## 2. list

内部数据结构：**环状双向链表**。
不能随机访问元素，可双向遍历。增加任何元素都不会使迭代器失效。删除元素时，除了指向当前被删除元素的迭代器外，其它迭代器都不会失效。

## 3. deque

内部数据结构：**数组**。
deque 是支持向两端高效地插入数据、支持随机访问的容器。deque 的数据被表示为一个分段数组，容器中的元素分段存放在一个个大小固定的数组中，此外容器还需要维护一个存放这些数组首地址的索引数组。

总结：
1、如果你需要高效的随即存取，而不在乎插入和删除的效率，使用 vector；
2、如果你需要大量的插入和删除，而不关心随即存取，则应使用 list；
3、如果你需要随即存取，而且关心两端数据的插入和删除，则应使用 deque。

## 4. stack

内部数据结构：底层以某种容器（ **list 或 deque（缺省）**）作为数据结构。
stack 是一种先进后出（FILO）的数据结构。它只有一个出口，stack 允许新增元素，移除元素，取得最顶端元素。但除了最顶端外，没有任何其它方法可以存取stack的其它元素，stack不允许遍历行为。
缺省情况下以 deque 作为底部数据结构，并封闭其头端开口，便能实现一个 deuqe。

## 5. queue

内部数据结构：底层以某种容器（ **list 或 deque（缺省）**）作为数据结构。
queue 是一种先进先出（First In First Out,FIFO）的数据结构。它有两个出口，queue 允许新增元素，移除元素，从最底端加入元素，取得最顶端元素。但除了最底端可以加入，最顶端可以取出外，没有任何其它方法可以存取 queue 的其它元素。
缺省情况下以 deque 作为底部数据结构，并封闭其头端入口和尾端出口，便能实现一个 queue。

stack 和 queue 其实是适配器，而不叫容器，因为是对容器的再封装。

## 6. set和multiset

内部数据结构：**RB-tree**。
set 底层是通过红黑树（RB-tree）来实现的，由于红黑树是一种平衡二叉搜索树，自动排序的效果很不错，所以标准的 STL 的 set 即以 RB-Tree 为底层机制。又由于 set 所开放的各种操作接口，RB-tree 也都提供了，所以几乎所有的 set 操作行为，都只有转调用 RB-tree 的操作行为而已。
multiset的特性以及用法和 set 完全相同，唯一的差别在于它允许键值重复，因此它的插入操作采用的是底层机制是 RB-tree 的 insert_equal() 而非 insert_unique()。

## 7. map和multimap

内部数据结构：**RB-tree**。
与set的结构一样，都是使用 RB-tree 实现的，只不过map的所有元素都是 pair，同时拥有实值（value）和键值（key）。pair 的第一元素被视为键值，第二元素被视为实值。
multimap 的特性以及用法与 map 完全相同，唯一的差别在于它允许键值重复，因此它的插入操作采用的是底层机制 RB-tree 的 insert_equal() 而非 insert_unique。

## 8. unordered_set和unordered_multiset

内部数据结构：**hash table**。
底层由哈希表实现。无序，查找元素的时间复杂度为常数。

## 9. unordered_map和unordered_multimap

内部数据结构：**hash table**。
底层由哈希表实现。无序，查找元素的时间复杂度为常数。
