---
title: 二叉树的前中后序递归法和迭代法
date: 2021-02-04 11:57:25
tags: DataStructure
categories:
  - DataStructure
copyright: true
---

# 二叉树的前中后序递归法和迭代法

二叉树根据遍历方式，分为**深度优先搜索（DFS）**和**广度优先搜索（BFS）**。

二叉树的深度优先搜索：

- 前序遍历
- 中序遍历
- 后序遍历

二叉树的广度优先搜索：

- 层序遍历
<!-- more -->

![binary-tree-traversal.png](https://i.loli.net/2021/02/04/e1dnxwVrZ2oLyEu.png)

以上二叉树中，深度优先搜索结果：

- 前序遍历（**中**左右）：1 2 4 5 8 3 6 7 9
- 中序遍历（左**中**右）：4 2 5 8 1 6 3 9 7
- 后续遍历（左右**中**）：4 8 5 2 6 9 7 3 1

广度优先搜索结果：

- 层序遍历：1 2 3 4 5 6 7 8 9

## 递归法

递归算法三要素：

1. 确定递归的返回值和参数：

   确定哪些参数是递归的过程中需要处理的，那么就在递归函数里加上这个参数， 并且还要明确每次递归的返回值是什么进而确定递归函数的返回类型。

2. 确定递归的终止条件：

   操作系统通过栈结构来存储每一层递归的信息，而栈空间是有限的，如果递归没有终止，则会一直递归下去，操作系统的内存栈必然会溢出，导致程序崩溃。

   所以何时终止？如何终止？这都是必须要考虑到的。

3. 确定单层递归的处理逻辑：

   确定每一层递归需要处理的信息。在这里也就会重复调用自己来实现递归的过程。

下面以**返回前序遍历的数组**举例：

```C++
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int v) : val(v), left(nullptr), right(nullptr) {}
}
```

1. 确定递归的返回值和参数：要求返回前序遍历的数组，所以递归函数不需要返回值，参数里需要传入二叉树的当前节点和 vector 用于存放遍历的数组。

   ```c++
   void preOrder(TreeNode* node, vector<int>& vec)
   ```

2. 确定递归的终止条件:在递归过程中，为了防止无限递归下去，需要一个终止条件，本例子中，当前节点为空时，不需要（也不能）处理任何信息，所以直接返回就可以了。

   ```c++
   if (!node) return;
   ```

3. 确定单层递归的处理逻辑:前序遍历是以`中->左->右`来进行遍历的，所以优先处理中节点信息，然后再往左节点和右节点递归。

   ```c++
   vec.push_back(node->val);
   traversal(node->left, vec);
   traversal(node->right, vec);
   ```

根据以上要素，完成前序遍历的完整代码如下：

```C++
class Solution {
    void traversal(TreeNode* node, vector<int>& vec) {
        if (!node) return;

        vec.push_back(node->val);
        traversal(node->left, vec);
        traversal(node->right, vec);
    }
public:
    vector<int> preOrderTraversal(TreeNode* root) {
        vector<int> result;
        traversal(root, result);
        return result;
    }
};
```

根据三要素的思想，不难写出中序遍历和后续遍历，如下：

中序遍历：

```c++
void traversal(TreeNode* node, vector<int>& vec) {
    if (!node) return;

    traversal(node->left, vec);
    vec.push_back(node->val);
    traversal(node->right, vec);
}
```

后续遍历：

```c++
void traversal(TreeNode* node, vector<int>& vec) {
    if (!node) return;

    traversal(node->left, vec);
    traversal(node->right, vec);
    vec.push_back(node->val);
}
```

## 迭代法

递归法的实现就是每一次递归调用都会把函数的局部变量，参数值和返回地址等压入调用栈中，然后递归返回的时候，从栈顶弹出上一次递归的各项参数，所以这就是递归为什么可以返回上一层位置的原因。

所以通过栈结构，无需递归也可以实现二叉树的前中后序遍历。

### 前序遍历

前序遍历就是优先处理中间节点，然后再进行左右节点的处理。所以入栈顺序应该是先压入根节点，接着将右节点压入栈中，再加入左节点。

```c++
class Solution {
public:
    vector<int> preOrderTraversal(TreeNode* root) {
        vector<int> result;
        if (!root) return result;
        stack<TreeNode*> st;	// 辅助栈
        st.push(root);
        while (!st.empty()) {
            TreeNode* node = st.top();
            st.pop();
            result.push_back(node->val);	// 中节点处理
            // 根据栈的特性（先进后出），需要先压入右节点
            if (node->right)
                st.push(node->right);		// 右节点
            if (node->left)
                st.push(node->left);		// 左节点
        }
        return result;
    }
};
```

### 中序遍历

中序遍历是左中右，先访问的是二叉树顶部的节点，然后一层一层向下访问，直到到达树左面的最底部，再开始处理节点（也就是在把节点的数值放进 result 数组中），这就造成了处理顺序和访问顺序是不一致的。

那么在使用迭代法写中序遍历，就需要借用指针的遍历来帮助访问节点，栈则用来处理节点上的元素。

```c++
class Solution {
public:
    vector<int> midOrderTraversal(TreeNode* root) {
        vector<int> result;
        stack<TreeNode*> st;    // 辅助栈
        TreeNode* cur = root;
        while (cur || !st.empty()) {
            if (cur) {  // 先访问到二叉树的最底层
                st.push(cur);   // 将访问到的节点压入栈中
                cur = cur->left;            // 左节点
            } else {    // 此时当前节点为空（无左节点）
                cur = st.top();	// 从栈里弹出节点
                st.pop();
                result.push_back(cur->val); // 中节点的数据处理
                cur = cur->right;           // 右节点
            }
        }
        return result;
    }
};
```

### 后序遍历

前序遍历的遍历顺序是：`左右中`，后序遍历的顺序是：`中左右`。

那么我只需要在前序遍历时，将`左右中`改为`右左中`，然后再将数组反转（`中左右`），即可得到后序遍历的结果。

```c++
class Solution {
public:
    vector<int> postOrderTraversal(TreeNode* root) {
        vector<int> result;
        if (!root) return result;
        stack<TreeNode*> st;    // 辅助栈
        st.push(root);
        while (!st.empty()) {
            TreeNode* node = st.top();
            st.pop();
            result.push_back(node->val);    // 中节点处理
            // 此时先压入左节点，将前序遍历改为 右左中
            if (node->left)
                st.push(node->left);        // 左节点
            if (node->right)
                st.push(node->right);       // 右节点
        }
        reverse(result.begin(), result.end());
        return result;
    }
};
```

## 迭代方式统一写法

因为迭代法实现的前中后序风格不统一，除了前序和后序有关联，中序完全就是另一种写法了。

针对三种遍历方式，使用迭代法是可以写出统一风格的代码的。

不统一的问题是：无法同时解决访问节点和处理节点不一致的情况。

那我们就将访问的节点放入栈中，把要处理的节点也放入栈中但是要做标记。

如何标记呢？**就是要处理的节点放入栈之后，紧接着放入一个空指针作为标记。** 这种方法也可以叫做标记法。

完整代码如下：

### 前序遍历

```c++
class Solution {
public:
    vector<int> preOrderTraversal(TreeNode* root) {
        vector<int> result;
        if (!root) return result;
        stack<TreeNode*> st;
        st.push(root);
        while (!st.empty()) {
            TreeNode* node = st.top();
            if (!node) {
                st.pop();   // 将该节点弹出，避免重复操作，下面再将右左中节点添加到栈中

                if (node->right) st.push(node->right);  // 添加右节点

                if (node->left) st.push(node->left);    // 添加左节点

                st.push(node);                          // 添加中节点
                st.push(nullptr);   // 中节点访问过，但是还没有处理，加入空节点做为标记。
            } else {    // 只有遇到空节点的时候，才将下一个节点放进结果集
                st.pop();   // 将空节点弹出
                node = st.top();    // 重新取出栈中元素
                st.pop();
                result.push_back(node->val);    // 处理节点
            }
        }
        return result;
    }
};
```

### 中序遍历

```c++
class Solution {
public:
    vector<int> midOrderTraversal(TreeNode* root) {
        vector<int> result;
        if (!root) return result;
        stack<TreeNode*> st;
        st.push(root);
        while (!st.empty()) {
            TreeNode* node = st.top();
            if (!node) {
                st.pop();

                if (node->right) st.push(node->right);  // 添加右节点

                st.push(node);                          // 添加中节点
                st.push(nullptr);

                if (node->left) st.push(node->left);    // 添加左节点
            } else {
                st.pop();
                node = st.top();
                st.pop();
                result.push_back(node->val);
            }
        }
        return result;
    }
};
```

### 后序遍历

```c++
class Solution {
public:
    vector<int> postOrderTraversal(TreeNode* root) {
        vector<int> result;
        if (!root) return result;
        stack<TreeNode*> st;
        st.push(root);
        while (!st.empty()) {
            TreeNode* node = st.top();
            if (!node) {
                st.pop();

                st.push(node);                          // 添加中节点
                st.push(nullptr);

                if (node->right) st.push(node->right);  // 添加右节点

                if (node->left) st.push(node->left);    // 添加左节点
            } else {
                st.pop();
                node = st.top();
                st.pop();
                result.push_back(node->val);
            }
        }
        return result;
    }
};
```
