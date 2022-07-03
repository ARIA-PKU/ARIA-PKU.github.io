---
title: 剑指offer2
date: 2022-06-20T13:04:08+08:00
lastmod: 2022-06-20T13:04:08+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/6_15/sky.jpg

categories:
  - 算法题整理
tags:
  - 剑指offer
# nolastmod: true
draft: false
---

剑指offer里面的一些有趣的题目

<!--more-->

# [剑指 Offer 03. 数组中重复的数字](https://leetcode.cn/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/)

**题意：**

找出数组中重复的数字。


在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

**思路：**

时间复杂度O(N)，空间复杂度O(1)

利用下标，如果当前点在对应的下标，则遍历下一个点；

如果当前点和其对应下标点的值相同，则找到相同值；

如果不同则交换，因为这里数字范围，一定能交换到最终位置。

最终，没有相同元素返回-1。

```
class Solution {
public:
    int findRepeatNumber(vector<int>& nums) {
        for (uint32_t i = 0; i < nums.size();) {
            if (nums[i] == i) i ++;
            else if (nums[i] == nums[nums[i]]) return nums[i];
            else swap(nums[i], nums[nums[i]]);
        }
        return -1;
    }
};
```

# 不修改数组找出重复的数字

题目：https://www.acwing.com/problem/content/15/

思路：

利用抽屉原理和二分的思想

每次找到一个范围中数字的个数，如果个数大于该范围，则必然有重复值，二分缩小空间。

时间复杂度为O(nlogn)，空间复杂度为O(1)

```
class Solution {
public:
    int duplicateInArray(vector<int>& nums) {
        int l = 1, r = nums.size() - 1;
        while (l < r) {
            int mid = l + r >> 1;
            int s = 0;
            for (auto& x: nums) s += (x >= l && x <= mid);
            if (s > mid - l + 1) r = mid;
            else l = mid + 1;
        }
        return r;
    }
};
```

# [剑指 Offer 04. 二维数组中的查找](https://leetcode.cn/problems/er-wei-shu-zu-zhong-de-cha-zhao-lcof/)

**题意：**

在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个高效的函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

**思路：**

类似二分的思想，比较巧妙，但是见过一次就很容易记住

```
class Solution {
public:
    bool findNumberIn2DArray(vector<vector<int>>& matrix, int target) {
        int m = matrix.size();
        if (!m) return false;
        int n = matrix[0].size();
        int i = m - 1, j = 0;
        while (i >= 0 && j < n) {
            if (matrix[i][j] == target) return true;
            else if (matrix[i][j] > target) i --;
            else j ++;
        }
        return false;
    }
};
```

# 替换空格

题目：https://www.acwing.com/problem/content/17/

思路：

从后向前添加

```
class Solution {
public:
    string replaceSpaces(string &str) {
        int len = str.size(), n = len;
        for (auto& c: str)
            if (c == ' ') len += 2;
        
        str.resize(len);
        for (int i = n - 1, j = len - 1; i >= 0; i --) {
            if (str[i] == ' ') {
                str[j --] = '0';
                str[j --] = '2';
                str[j --] = '%';
            } else {
                str[j --] = str[i];
            }
        }
        return str;
    }
};
```

# 重建二叉树

题目：https://www.acwing.com/problem/content/23/

题意：输入一棵二叉树前序遍历和中序遍历的结果，请重建该二叉树。

思路：

分治递归建树

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    vector<int> pre, in;
    TreeNode* create(int pl, int pr, int il, int ir) {
        if (pl > pr) return NULL;
        
        int u = pre[pl], i;
        TreeNode* root = new TreeNode(u);
        for (i = il; i <= ir; i ++)
            if (in[i] == u) break;
        int k = i - il;
        root->left = create(pl + 1, pl + k, il, i - 1);
        root->right = create(pl + k + 1, pr, i + 1, ir);
        return root;
    }
    
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        int n = preorder.size();
        pre = preorder, in = inorder;
        return create(0, n - 1, 0, n - 1);
    }
};
```

# 二叉树的下一个节点

题目：https://www.acwing.com/problem/content/31/

题意：

给定一棵二叉树的其中一个节点，请找出中序遍历序列的下一个节点。

思路：

分两种情况：

1、如果有右儿子，则后继节点为右儿子的最左子节点

2、没有右儿子，如果当前节点是某个节点的左儿子的最右节点，则需要向上找到这个节点，返回。类型下图的结构（灵魂构图）：

```
	x(target)
x
   	x
   		...
   			x(cur)
```

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode *father;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL), father(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* inorderSuccessor(TreeNode* p) {
        // 从右儿子的后继中找
        if (p->right) {
            p = p->right;
            while (p->left) p = p->left;
            return p;
        }
        // 找上面的存在左儿子的分支节点
        while (p->father && p == p->father->right)
            p = p->father;
        
        return p->father;
    }
};
```

# 旋转数组的最小数字

**题目：**https://www.acwing.com/problem/content/20/

**题意：**

把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。

输入一个升序的数组的一个旋转，输出旋转数组的最小元素。

例如数组 {3,4,5,1,2}为 {1,2,3,4,5} 的一个旋转，该数组的最小值为 1。

数组可能包含重复项。

**思路：**

有其他类似的题目，不一样的地方在于会有相同元素，这里要增加一步对结尾元素的预处理，这样最坏情况（所有元素都相等）会是O(N)的，平均是O（logN)的时间复杂度。

```
class Solution {
public:
    int findMin(vector<int>& nums) {
        int l = 0, r = nums.size() - 1;
        if (r < 0) return -1;
        // 处理最后的元素与开头元素相同的情况
        while (r > 0 && nums[r] == nums[l]) r --;
        if (nums[r] >= nums[0]) return nums[0];
        // 接下的是左、右部分全都有序
        while (l < r) {
            int mid = l + r >> 1;
            if (nums[mid] >= nums[0]) l = mid + 1;
            else r = mid;
        }
        return nums[r];
    }
};
```

# 矩阵中的路径

题目：https://www.acwing.com/problem/content/description/21/

题意：

请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。

路径可以从矩阵中的任意一个格子开始，每一步可以在矩阵中向左，向右，向上，向下移动一个格子。

如果一条路径经过了矩阵中的某一个格子，则之后不能再次进入这个格子。

思路：

dfs+回溯

```
class Solution {
public:
    int m, n;
    vector<vector<bool>> st;
    vector<vector<char>> mat;
    string s;
    int dx[4] = {1, 0, -1, 0}, dy[4] = {0, 1, 0, -1};
    bool dfs(int x, int y, int u) {
        if (mat[x][y] != s[u]) return false;
        // 这里如果把下面这句提前并改成u == s.size()会出问题，以后需要注意一下这一点
        if (u == s.size() - 1) return true;
        
        st[x][y] = true;
        for (int i = 0; i < 4; i ++) {
            int nx = x + dx[i], ny = y + dy[i];
            if (nx >= 0 && nx < m && ny >= 0 && ny < n && !st[nx][ny])
                if (dfs(nx, ny, u + 1))
                    return true;
        }
        st[x][y] = false;
        return false;
    }
    
    bool hasPath(vector<vector<char>>& matrix, string &str) {
        m = matrix.size();
        if (m == 0) return false;
        n = matrix[0].size();
        st.resize(m, vector<bool>(n, false));
        mat = matrix;
        s = str;
        for (int i = 0; i < m; i ++)
            for (int j = 0; j < n; j ++)
                if (dfs(i, j, 0))
                    return true;
        
        return false;
    }
};
```

# 机器人的运动范围

题目：https://www.acwing.com/problem/content/22/

题意：

地上有一个 mm 行和 nn 列的方格，横纵坐标范围分别是 0∼m−10∼m−1 和 0∼n−10∼n−1。

一个机器人从坐标 (0,0)(0,0) 的格子开始移动，每一次只能向左，右，上，下四个方向移动一格。

但是不能进入行坐标和列坐标的数位之和大于 kk 的格子。

请问该机器人能够达到多少个格子

思路：

典型的BFS

```
#define x first
#define y second
class Solution {
public:
    typedef pair<int, int> PII;
    bool check(int a, int b, int k) {
        int sum = 0;
        while (a) {
            sum += a % 10;
            a /= 10;
        }
        while (b) {
            sum += b % 10;
            b /= 10;
        }
        if (sum <= k) return true;
        return false;
    }
    int movingCount(int k, int m, int n)
    {
        if (m == 0 || n == 0) return 0;
        int res = 0;
        vector<vector<bool>> st(m, vector<bool>(n, false));
        queue<PII> q;
        q.push({0, 0});
        st[0][0] = true;
        int dx[4] = {0, 1, 0, -1}, dy[4] = {1, 0, -1, 0};
        while (q.size()) {
            auto u = q.front();
            q.pop();
            res ++;
            int a = u.x, b = u.y;
          
            for (int i = 0; i < 4; i ++) {
                int nx = a + dx[i], ny = b + dy[i];
                if (nx >= 0 && nx < m && ny >= 0 && ny < n && !st[nx][ny]) {
                    if (check(nx, ny, k)) {
                        // 需要在放入队列之前置为true
                        st[nx][ny] = true;
                        q.push({nx, ny});
                    }
                }
            }
        }
        return res;
    }
};
```

# 剪绳子

**题目**：https://www.acwing.com/problem/content/24/

**题意：**

给你一根长度为 n 绳子，请把绳子剪成 m 段

每段的绳子的长度可能的最大乘积是多少？

例如当绳子的长度是 8 时，我们把它剪成长度分别为 2、3、3 的三段，此时得到最大的乘积 18。

**思路：**

这道题目是数学中一个很经典的问题。

下面我们给出证明：

首先把一个正整数 NN 拆分成若干正整数只有有限种拆法，所以存在最大乘积。

假设 N=n1+n2+…+nk，并且 n1×n2×…×nk 是最大乘积。

显然1不会出现在其中；

如果对于某个 ii 有 ni≥5ni≥5，那么把 nini 拆分成 3+(ni−3)3+(ni−3)，我们有 3(ni−3)=3ni−9>ni

如果 ni=4ni=4，拆成 2+22+2乘积不变，所以不妨假设没有4；

如果有三个以上的2，那么 3×3>2×2×2，所以替换成3乘积更大；

综上，选用尽量多的3，直到剩下2或者4时，用2。

时间复杂度分析：当 n 比较大时，n 会被拆分成 ⌈n/3⌉ 个数，我们需要计算这么多次减法和乘法，所以时间复杂度是 O(n)。

```
class Solution {
public:
    int maxProductAfterCutting(int n) {
        if (n < 4) return n - 1;
        int res = 1;
        // 这里是大于4，因为是4的时候，切分成2*2相当于不变
        while (n > 4) {
            n -= 3;
            res *= 3;
        }
        res *= n;
        return res;
    }
};
```

# 数值的整数次方

题目：https://www.acwing.com/problem/content/description/26/

题意：

实现函数*double Power(double base, int exponent)*，求*base*的 *exponent*次方。

不得使用库函数，同时不需要考虑大数问题。

只要输出结果与答案的绝对误差不超过 10−210−2 即视为正确。

思路：

快速幂问题

```
class Solution {
public:
    double Power(double base, int n) {
       bool isN = false;
       if (n < 0) {
           isN = true;
           n *= -1;
       }
       double res = 1.0;
       for (int i = 0; i < 32; i ++) {
           if (n >> i & 1) res *= base;
           base *= base;
       }
 
       if (isN) return 1 / res;
       return res;
    }
};
```

# 在O(1)时间删除链表结点

题目：https://www.acwing.com/problem/content/description/85/

题意：

给定单向链表的一个节点指针，定义一个函数在O(1)时间删除该结点。

假设链表一定存在，并且该节点一定不是尾节点。

思路：

算是脑经急转弯，把当前点的值修改为后一个点的值，然后删除后一个点

```
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    void deleteNode(ListNode* node) {
        auto p = node->next;
        node->val = p->val;
        node->next = p->next;
        delete p;
    }
};
```

# 删除链表中重复的节点

题目：https://www.acwing.com/problem/content/description/27/

题意：

在一个排序的链表中，存在重复的节点，请删除该链表中重复的节点，重复的节点不保留。

思路：

重复节点不保留，通过节点比较确定是否保留当前节点。

```
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode* deleteDuplication(ListNode* head) {
        ListNode* dummy = new ListNode(0);
        dummy->next = head;
        auto u = dummy;
        while (u->next) {
            auto p = u->next;
            while (p->next && p->next->val == p->val) p = p->next;
            // 如果不存在重复的元素，则直接遍历下一个位置
            if (u->next == p) u = u->next;
            // 否则将next定为下一个元素的开头
            else u->next = p->next;
        }
        return dummy->next;
    }
};
```

# 正则表达式匹配

题目：https://www.acwing.com/problem/content/description/28/

题意：

请实现一个函数用来匹配包括`'.'`和`'*'`的正则表达式。

模式中的字符`'.'`表示任意一个字符，而`'*'`表示它前面的字符可以出现任意次（含0次）。

思路：

关于下面的当前p为\*，且非0个的匹配的情况举例说明：

s = "abccc", p = "abc\*"的情况，

首先，p的最后一个字母和s的最后一个字母可以匹配，则转移到了s = "abcc"和 p = “abc\*"的情况，

同理，可以再转移到s = "abc"和 p = “abc\*"，继续转移到s = "ab"和 p = “abc\*"，

这里我们就可以由*代表0个c，转移得到s="ab"和p=”ab“的情况，得到其相同。

因此，正向过程可以得到其是匹配的。

```
class Solution {
public:
    bool check(string& s, string& p, int i, int j) {
        if (i == 0) return false;
        if (p[j - 1] == '.' || s[i - 1] == p[j - 1]) return true;
        return false;
    }
    bool isMatch(string s, string p) {
        int m = s.size(), n = p.size();
        vector<vector<int>> f(m + 1, vector<int>(n + 1, false));
        f[0][0] = true;
        for (int i = 0; i <= m; i ++) {
            for (int j = 1; j <= n; j ++) {
                // 非*的情况很容易理解，相同则继承前面的关系
                if (p[j - 1] != '*') {
                    if (check(s, p, i, j))
                        f[i][j] = f[i - 1][j - 1];
                } else {
                    // 首先，将*代表0个情况考虑进去
                    f[i][j] |= f[i][j - 2];
                    // 然后，这里判断的是s的最后一个字母和p的最后一个是否能够匹配
                    // 如果可以匹配，则转移到了f[i - 1][j]的情况
                    if (check(s, p, i, j - 1))
                        f[i][j] |= f[i - 1][j];
                }
            }
        }
        return f[m][n];
    }
};
```

# 表示数值的字符串

题目：https://www.acwing.com/problem/content/29/

思路：

模拟题

```
class Solution {
public:
    bool isNumber(string s) {
        bool isNum = false;
        int i = 0;
        // 处理前导空格
        while (s[i] == ' ') i ++;
        // 处理最开始的正负号
        if (s[i] == '+' || s[i] == '-') i ++;
        // 数字和小数点
        while (s[i] >= '0' && s[i] <= '9') {
            i ++;
            isNum = true;
        }
        if (s[i] == '.') i ++;
        while (s[i] >= '0' && s[i] <= '9') {
            i ++;
            isNum = true;
        }
        // 如果有e或E，先判断前面的是否是数字
        if (isNum && (s[i] == 'e' || s[i] == 'E')) {
            i ++;
            isNum = false;
            if (s[i] == '+' || s[i] == '-') i ++;
            while (s[i] >= '0' && s[i] <= '9') {
                i ++;
                isNum = true;
            }
        }
        return i == s.size() && isNum;
    }
};
```

# 调整数组顺序使奇数位于偶数前面

题目：https://www.acwing.com/problem/content/30/

思路：

双指针，奇数跳过，偶数，交换再判断

```
class Solution {
public:
    void reOrderArray(vector<int> &array) {
         int l = 0, r = array.size() - 1;
         while (l < r) {
             if (array[l] % 2) l ++;
             else swap(array[l], array[r --]);
         }
    }
};
```

# 树的子结构

题目：https://www.acwing.com/problem/content/description/35/

题意：

输入两棵二叉树 A，B，判断 B 是不是 A 的子结构。

我们规定空树不是任何树的子结构。

思路：

递归。递归地将A中的每一个节点与B的整个结构进行比较；在比较过程中要递归遍历A和B的每一个对应节点进行比较。

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    bool hasSubtree(TreeNode* r1, TreeNode* r2) {
        if (!r1 || !r2) return false;
        return isSame(r1, r2) || hasSubtree(r1->left, r2) || hasSubtree(r1->right, r2);
    }
    bool isSame(TreeNode* r1, TreeNode* r2) {
        if (!r2) return true;
        if (!r1 || r1->val != r2->val) return false;
        return isSame(r1->left, r2->left) && isSame(r1->right, r2->right);
    }
};
```

# 二叉树的镜像

题目：https://www.acwing.com/problem/content/37/

题意：输入一个二叉树，将它变换为它的镜像。

思路：

简单dfs

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    void dfs(TreeNode*& root) {
        auto u = root->right;
        root->right = root->left;
        root->left = u;
        if (root->left) dfs(root->left);
        if (root->right) dfs(root->right);
        return;
    }
    void mirror(TreeNode* root) {
        if (!root) return;
        dfs(root);
        return;
    }
};
```

# 对称的二叉树

题目：https://www.acwing.com/problem/content/38/

题意：如果一棵二叉树和它的镜像一样，那么它是对称的。

思路：

简单dfs

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    bool isSame(TreeNode* l, TreeNode* r) {
        if (!l && !r) return true;
        if (!l || !r) return false;
        return l->val == r->val && isSame(l->left, r->right) && isSame(r->left, l->right);
    }
    bool isSymmetric(TreeNode* root) {
        if (!root) return true;
        return isSame(root->left, root->right);
    }
};
```

# 包含min函数的栈

题目：https://www.acwing.com/problem/content/90/

思路：

利用pair存储，第一个存储当前元素，第二个存储当前最小元素

```
#define x first
#define y second

typedef pair<int, int> PII;

class MinStack {
public:
    /** initialize your data structure here. */
    stack<PII> st;
    MinStack() {
    }
    
    void push(int x) {
        if (st.size() && x > st.top().y) st.push({x, st.top().y});
        else st.push({x, x});
    }
    
    void pop() {
        st.pop();
    }
    
    int top() {
        auto u = st.top();
        return u.x;
    }
    
    int getMin() {
        auto u = st.top();
        return u.y;
    }
};

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack obj = new MinStack();
 * obj.push(x);
 * obj.pop();
 * int param_3 = obj.top();
 * int param_4 = obj.getMin();
 */
```

# 栈的压入、弹出序列

题目：https://www.acwing.com/problem/content/40/

题意：

输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否可能为该栈的弹出顺序。

思路：

模拟题

```
class Solution {
public:
    bool isPopOrder(vector<int> A,vector<int> B) {
        if (A.size() != B.size()) return false;
        stack<int> st;
        int k = 0;
        for (uint32_t i = 0; i < A.size(); i ++) {
            st.push(A[i]);
            while (!st.empty() && B[k] == st.top()) {
                st.pop();
                k ++;
            }
        }
        return st.empty();
    }
};
```

# 二叉搜索树的后序遍历序列

题目：https://www.acwing.com/problem/content/description/44/

题意：

输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历的结果。

假设输入的数组的任意两个数字都互不相同。

思路：

利用后序性质，递归遍历即可。

```
class Solution {
public:
    // 后序遍历，前部分小于最后的值，后部分大于最后的值
    bool verifySequenceOfBST(vector<int> s) {
        int n = s.size();
        return check(s, 0, n - 1);
    }
    bool check(vector<int>& s, int l, int r) {
        if (l >= r) return true;
        int u = s[r], ul = l;
        while (s[ul] < u) ul ++;
        int p = l - 1;
        while (ul < r) { 
            if (s[ul] < u) return false;
            ul ++;
        }
        return check(s, l, p) && check(s, p + 1, r - 1);
    }
};
```

# 二叉树中和为某一值的路径

题目：https://www.acwing.com/problem/content/45/

题意：

输入一棵二叉树和一个整数，打印出二叉树中结点值的和为输入整数的所有路径。

从树的根结点开始往下一直到叶结点所经过的结点形成一条路径。

保证树中结点值均不小于 0。

思路：

dfs+回溯，注意剪枝，能节省很多时间。

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    int sum;
    void dfs(TreeNode* root, int u) {
    	// 剪枝
        if (u > sum) return;
        
        path.push_back(root->val);
        u += root->val;
        
        if (!root->left && !root->right && u == sum) ans.push_back(path);
        if (root->left) dfs(root->left, u);
        if (root->right) dfs(root->right, u);
        
        u -= root->val;
        path.pop_back();
    }
    vector<vector<int>> findPath(TreeNode* root, int sum) {
        if (!root) return ans;
        this->sum = sum;
        dfs(root, 0);
        return ans;
    }
};
```

# 二叉搜索树与双向链表

题目：https://www.acwing.com/problem/content/87/

题意：输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。

思路：

递归：

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode *head = NULL, *pre = NULL; 
    TreeNode* convert(TreeNode* root) {
        dfs(root);
        return head;
    }
    void dfs(TreeNode* root) {
        if (!root) return;
        dfs(root->left);
        if (pre == NULL) head = root;
        else {
            pre->right = root;
            root->left = pre;
        }
        pre = root;
        dfs(root->right);
    }
};
```

迭代：

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* convert(TreeNode* root) {
        stack<TreeNode*> q;
        auto u = root;
        TreeNode *head = NULL, *pre = NULL;
        while (q.size() || u) {
            while (u) {
                q.push(u);
                u = u->left;
            }
            u = q.top();
            q.pop();
            if (!pre) head = u;
            else {
                pre->right = u;
                u->left = pre;
            }
            pre = u;
            u = u->right;
        }
        return head;
    }
};
```

# 序列化二叉树

题目：https://www.acwing.com/problem/content/description/46/

题意：

请实现两个函数，分别用来序列化和反序列化二叉树。

思路：

使用递归实现起来确实简便很多。

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    
    void dfs_s(TreeNode* root, string& res) {
        if (!root) {
            res += "null,";
            return;
        }
        else {
            res += to_string(root->val) + ","; 
            dfs_s(root->left, res);
            dfs_s(root->right, res);
        }
    }
    
    // Encodes a tree to a single string.
    string serialize(TreeNode* root) {
        string res = "";
        dfs_s(root, res);
        return res;
    }

    TreeNode* dfs_d(string& data, int& u) {
        int k = u;
        while (data[k] != ',') k ++;
        if (data[u] == 'n') {
            u = k + 1;
            return NULL;
        }
        int val = 0, sign = 1;
        if (u < k && data[u] == '-') sign = -1, u ++;
        for (int i = u; i < k; i ++) val = val * 10 + data[i] - '0';
        val *= sign;
        u = k + 1;
        auto root = new TreeNode(val);
        root->left = dfs_d(data, u);
        root->right = dfs_d(data, u);
        return root;
    }
    // Decodes your encoded data to tree.
    TreeNode* deserialize(string data) {
        int u = 0;
        return dfs_d(data, u);
    }
};
```

# 数组中出现次数超过一半的数字

题目：https://www.acwing.com/problem/content/48/

思路：

摩尔投票法思路，还有一个类似的求众数的是求超过三分之一的：

多数元素II: https://leetcode.cn/problems/majority-element-ii/

如果不一定有结果的话，可以再从头遍历一遍校验。

```
class Solution {
public:
    int moreThanHalfNum_Solution(vector<int>& nums) {
        int cur = nums[0], cnt = 0;
        for (auto& x: nums) {
            if (x == cur) cnt ++;
            else cnt --;
            if (cnt <= 0) {
                cur = x;
                cnt = 1;
            }
        }
        return cur;
    }
};
```

# 从1到n整数中1出现的次数

题目：https://www.acwing.com/problem/content/51/

思路：

假设当前数字为abcdef，每个字母对应一位数字，

以c这一位为例分析，

1、前面的两位取00到ab-1，则c后面的三位def可以取000到999共1000位，总共ab*1000个1

2、当前面的两位取ab时，

​	1）如果c=0，则当前这位无法取1

​	2）如果c=1，则后面的可以取0到def，共def+1个1

​	3）如果c>1，则后面的可以取0到999，共1000位

时间复杂度是$log^2(N)$

```
class Solution {
public:
    int numberOf1Between1AndN_Solution(int n) {
        if (!n) return 0;
        vector<int> numbers;
        while (n) numbers.push_back(n % 10), n /= 10;
        int cnt = 0, len = numbers.size();
        for (int i = len - 1; i >= 0; i --) {
            int left = 0, right = 0, t = 1;
            for (int j = len - 1; j > i; j --) left = left * 10 + numbers[j];
            for (int j = i - 1; j >= 0; j --) right = right * 10 + numbers[j], t *= 10;
            cnt += left * t;
            if (numbers[i] == 1) cnt += right + 1;
            if (numbers[i] > 1) cnt += t;
        }
        return cnt;
    }
};
```

# 数据流中的中位数

题目：https://www.acwing.com/problem/content/88/

思路：

建一个大根堆存储较小的一半，建一个小根堆存储较大的一半。

1、每次放入数据先放到大根堆中；

2、比较大根堆和小根堆堆顶元素，如果较大部分的最小值小于较小部分的最大值，则必然是新加入的这个元素属于较大的一半，交换两个堆顶元素即可；

3、再判断两个堆个数大小，如果较小部分的个数比较大部分的个数多的个数大于1，则无法直接取得中间元素，所以将较小部分的堆顶元素放入较大部分即可。

4、中位数在奇数时，是较小部分的堆顶元素，偶数时，是两个堆顶的元素和的平均数。

```
class Solution {
public:
    priority_queue<int> max_heap;
    priority_queue<int, vector<int>, greater<int>> min_heap;
    
    void insert(int num){
        max_heap.push(num);
        if (min_heap.size() && min_heap.top() < max_heap.top()) {
            auto a = min_heap.top(), b = max_heap.top();
            min_heap.pop();
            max_heap.pop();
            max_heap.push(a), min_heap.push(b);
        }
        if (max_heap.size() > min_heap.size() + 1) {
            auto u = max_heap.top();
            max_heap.pop();
            min_heap.push(u);
        }
    }

    double getMedian(){
        if (min_heap.size() + max_heap.size() & 1) return max_heap.top();
        else return (min_heap.top() + max_heap.top()) / 2.0;
    }
};
```

# 数字序列中某一位的数字

题目：https://www.acwing.com/problem/content/52/

思路：

数位模拟，找规律题目。

先不考虑0，1到9一共9位，占9*1=9位；

10到99共90位，占90*2 = 180位；

同理，可以依次递增找到接下来的规律。

一般这种问题代入实际数字计算边界条件会更容易：

以5为例，当前起始值为1，1 + 5 / 1 - 1即为5，当前的位所在的数字的可以表示为base + $\lceil$n / i$\rceil$；

这里带入0，发现也能够满足要求，所以无需特判0的情况；

以13为例，13 - 1 * 9 = 4，则表示在两位数中的第四位，两位数的起始值为10，4 / 2 = 2，为第二个数的最后一位，如果是12，则是3 / 2 ，因为也是第二个数，所以考虑上取整， $\lceil$n / i$\rceil$ = $\lfloor$(n + i - 1) / i$\rfloor$ 

根据余数来看，当余数为0的时候代表的是最后一位即第i位，因此加一个特判。

现在就找到了哪一个数的第几位，然后依次除掉末尾的几位，输出当前位即可。时间复杂度为log(n)。

```
typedef long long LL;
class Solution {
public:
    int digitAtIndex(int n) {
    	// i记录当前数字有几位，num记录当前位的数字共有多少个
    	// base记录当前数字的起始数字大小
        LL i = 1, num = 9, base = 1;
        while (n > i * num) {
            n -= i * num;
            i += 1;
            num *= 10;
            base *= 10;
        }
        int u = base + (n + i - 1) / i - 1;
        int p = n % i ? n % i : i;
        for (int k = 0; k < i - p; k ++) u /= 10;
        return u % 10;
    }
};
```

# 把数组排成最小的数

题目：https://www.acwing.com/problem/content/54/

思路：

可以简单推导传递关系，排序即可。

传递性：

ab < ba, bc < cb => ac < ca

```
class Solution {
public:
    static bool cmp(string& a, string& b) {
        return a + b < b + a;
    }
    string printMinNumber(vector<int>& nums) {
        vector<string> q;
        for (auto& x: nums) q.push_back(to_string(x));
        sort(q.begin(), q.end(), cmp);
        string ans;
        for (auto& x: q) ans += x;
        return ans;
    }
};
```

# 把数字翻译成字符串

题目：https://www.acwing.com/problem/content/55/

思路：

类似斐波那契数列，对当前的数字f[n]，必然可以作为一位来翻译，这对应f[n - 1]的翻译方法；

如果可以作为两位来翻译，则对应了f[n - 2]的翻译方法。

```
class Solution {
public:
    int getTranslationCount(string s) {
        int n = s.size();
        vector<int> f(n + 1);
        f[0] = f[1] = 1;
        for (int i = 2; i <= n; i ++) {
            f[i] = f[i - 1];
            if ((s[i - 2] == '1') || (s[i - 2] == '2' && s[i - 1] >= '0' && s[i - 1] <= '5'))
                f[i] += f[i - 2];
        }
        return f[n];
    }
};
```

# 丑数

题目：https://www.acwing.com/problem/content/58/

题意：

我们把只包含质因子 2、3 和 5 的数称作丑数（Ugly Number）。

例如 6、8 都是丑数，但 14 不是，因为它包含质因子 7。

求第 n 个丑数的值。

思路：

多路归并的思路。维护i，j，k三个指针，取最小值加入队列，然后将对应指针加1，由于有可能对应多个指针，为了避免结果重复要全部判断一遍。

```
class Solution {
public:
    int getUglyNumber(int n) {
        if (n == 1) return n;
        int i = 0, j = 0, k = 0;
        vector<int> u(1, 1);
        while (--n) {
            int t = min(u[i] * 2, min(u[j] * 3, u[k] * 5));
            u.push_back(t);
            if (t == u[i] * 2) i ++;
            if (t == u[j] * 3) j ++;
            if (t == u[k] * 5) k ++;
        }
        return u.back();
    }
};
```

# 数组中只出现一次的两个数字

题目：https://www.acwing.com/problem/content/69/	

题意：

一个整型数组里除了两个数字之外，其他的数字都出现了两次。

请写程序找出这两个只出现一次的数字。

你可以假设这两个数字一定存在。

思路：

将所有数字异或，相同元素的异或为0，最终的异或结果为两个答案的异或结果；

然后通过lowbit取最低位的1（可以取任意位的1，lowbit计算简单），这就是两个答案不同的位之一；

然后按照和当前最低位1与的结果是否为0，将所有元素分为两类，两类中相同元素想与为0，剩下的就是两个所求元素。

```
class Solution {
public:
    vector<int> findNumsAppearOnce(vector<int>& nums) {
        int t = 0;
        for (auto& x: nums) t ^= x;
        int u = t & (-t);
        int res1 = 0, res2 = 0;
        for (auto& x: nums) {
            if (x & u) res1 ^= x;
            else res2 ^= x;
        }
        return {res1, res2};
    }
};
```

# 数组中唯一只出现一次的数字

题目：https://www.acwing.com/problem/content/70/

题意：

在一个数组中除了一个数字只出现一次之外，其他数字都出现了三次。

请找出那个只出现一次的数字。

你可以假设满足条件的数字一定存在。

- 如果要求只使用 O(n)O(n) 的时间和额外 O(1)O(1) 的空间，该怎么做呢？

思路：

比较常规的想法是遍历每一位，如果1的个数不是3的倍数，则这个1属于只出现一次的元素，0出现的次数是无需考虑的。

```
class Solution {
public:
    int findNumberAppearingOnce(vector<int>& nums) {
        int ans = 0;
        for (int i = 31; i >= 0; i --) {
            int cnt = 0;
            for (auto& x: nums) {
                if (x >> i & 1) cnt ++;
            }
            ans *= 2;
            if (cnt % 3) ans += 1;
        }
        return ans;
    }
};
```

比较难想而且不太好理解的是状态转移的方式：

```
class Solution {
public:
    int findNumberAppearingOnce(vector<int>& nums) {
        int ones = 0, twos = 0;
        for(auto& x: nums){
            ones = (ones ^ x) & ~twos;
            twos = (twos ^ x) & ~ones;
        }
        return ones;
    }
};
```

# 翻转单词顺序

题目：https://www.acwing.com/problem/content/73/

题意：

输入一个英文句子，**单词之间用一个空格隔开，且句首和句尾没有多余空格**。

翻转句子中单词的顺序，但单词内字符的顺序不变。

为简单起见，标点符号和普通字母一样处理。

例如输入字符串`"I am a student."`，则输出`"student. a am I"`。

思路：

常见的反转思路，先将每一个单词内部反转，再反转整个字符串。

```
class Solution {
public:
    void reverseSingleWord(string& s, int l, int r) {
        while (l < r) {
            swap(s[l ++], s[r --]);
        }
    }
    
    string reverseWords(string s) {
        int l = 0, n = s.size();
        for (int i = 0; i < n; i ++) {
            if (s[i] == ' ') {
                reverseSingleWord(s, l, i - 1);
                l = i + 1;
            }
        }
        reverseSingleWord(s, l, n - 1);
        reverse(s.begin(), s.end());
        return s;
    }
};
```

# 二叉搜索树的第k个结点

题目：https://www.acwing.com/problem/content/66/

题意：

给定一棵二叉搜索树，请找出其中的第 kk 小的结点。

你可以假设树和 kk 都存在，并且 1≤k≤1≤k≤ 树的总结点数。

思路：

直接dfs

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* ans;
    int cnt;
    void dfs(TreeNode* root) {
        if (!root) return;
        dfs(root->left);
        if (-- cnt == 0) {
            ans = root;
            return;
        }
        dfs(root->right);
    }
    
    TreeNode* kthNode(TreeNode* root, int k) {
        cnt = k;
        dfs(root);
        return ans;
    }
};
```

利用栈：

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* kthNode(TreeNode* root, int k) {
        stack<TreeNode*> st;
        auto u = root;
        while (st.size() || u) {
            while (u) {
                st.push(u);
                u = u->left;
            }
            u = st.top();
            st.pop();
            if (-- k == 0) return u;
            u = u->right;
        }
        return NULL;
    }
};
```

# 平衡二叉树

题目：https://www.acwing.com/problem/content/68/

题意：

输入一棵二叉树的根结点，判断该树是不是平衡二叉树。

如果某二叉树中任意结点的左右子树的深度相差不超过 11，那么它就是一棵平衡二叉树。

思路：

这题需要结合求深度的题目，根据当前左右节点深度，先判断当前节点是否满足，再递归左右节点。

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    int depth(TreeNode* root) {
        if (!root) return 0;
        return max(depth(root->left), depth(root->right)) + 1;
    }
    bool isBalanced(TreeNode* root) {
    	// 记得返回空为true
        if (!root) return true;
        int l = depth(root->left), r = depth(root->right);
        return abs(l - r) <= 1 && isBalanced(root->left) && isBalanced(root->right);
    }
};
```

# 求1+2+…+n

题目：https://www.acwing.com/problem/content/80/

题意：

求 1+2+…+n，要求不能使用乘除法、for、while、if、else、switch、case 等关键字及条件判断语句 (A?B:C)。

思路：

递归加短路判断

```
class Solution {
public:
    int getSum(int n) {
        int res = n;
        // 利用n > 0短路
        (n > 0) && (res += getSum(n - 1));
        return res;
    }
};
```

# 滑动窗口的最大值

题目：https://www.acwing.com/problem/content/75/

思路：

双端队列，队列中存放数组下标；

每次往队列里放置元素时候，先把队列中比它小的pop出来，因为比它小的不可能是当前结果；

再判断一次队头元素是否超过滑动窗口大小，是则把队头pop出来；

当前位置大于滑动窗口大小存放答案。

```
class Solution {
public:
    vector<int> maxInWindows(vector<int>& nums, int k) {
        vector<int> ans;
        deque<int> q;
        for (int i = 0; i < nums.size(); i ++) {
            while (q.size() && nums[i] > nums[q.back()]) q.pop_back();
            q.push_back(i);
            if (q.size() && i - q.front() >= k) q.pop_front();
            if (i >= k - 1) ans.push_back(nums[q.front()]);
        }
        return ans;
    }
};
```

# 骰子的点数

题目：https://www.acwing.com/problem/content/76/

题意：

请求出投掷 n 次，掷出 n∼6n 点分别有多少种掷法。

思路：

dp，f\[i]\[j]表示第i轮次，掷出结果j的方法次数。

每轮掷出的结果只有1到6，如果要掷出结果j，则只需要累加上一轮次j - k掷出的次数即可。

```
class Solution {
public:
    vector<int> numberOfDice(int n) {
        vector<vector<int>> f(n + 1, vector<int>(6 * n + 1));
        f[0][0] = 1;
        for (int i = 1; i <= n; i ++) {  // 枚举轮次
            for (int j = i; j <= 6 * i; j ++) {  // 枚举总和可能的结果
                for (int k = 1; k <= 6; k ++) {  // 枚举当前轮次的结果
                    if (j >= k)
                        f[i][j] += f[i - 1][j - k];
                }
            }
        }
        return vector<int>(f[n].begin() + n, f[n].end());
    }
};

```

# 扑克牌的顺子

题目：https://www.acwing.com/problem/content/77/

题意：

从扑克牌中随机抽 55 张牌，判断是不是一个顺子，即这 55 张牌是不是连续的。

思路：

五张牌先排序，跳过0的元素，然后判断是否相等，差值是否大于等于5。

```
class Solution {
public:
    bool isContinuous( vector<int> numbers ) {
        if (numbers.empty()) return false;
        sort(numbers.begin(), numbers.end());
        int st = 0, k = 0;
        while (numbers[k] == 0) st ++, k ++;
        while (k < 5) {
            if ((k > 1 && numbers[k] == numbers[k - 1]) || (numbers[k] - numbers[st] >= 5))
                return false;
            k ++;
        }
        return true;
    }
};
```

# 圆圈中最后剩下的数字

题目：https://www.acwing.com/problem/content/78/

思路：

约瑟夫问题，求环内最后剩余的数字。

可以采用递归的方式求解，已知函数返回的就是最后结果，只需要递归地找到其在上一轮递归中的结果的位置对应当前轮次的实际值，直到n=1的时候返回0.

```
class Solution {
public:
    int lastRemaining(int n, int m){
        if (n == 1) return 0;
        return (lastRemaining(n - 1, m) + m ) % n ;
    }
};
```

# 不用加减乘除做加法

题目：https://www.acwing.com/problem/content/81/

题意：

写一个函数，求两个整数之和，要求在函数体内不得使用 ＋、－、×、÷四则运算符号。

思路：

位运算相关的题目。

两数异或，同0异1，利用这个可以求得两数不带进位的结果；

两数相与，同1异0，利用它可以将每个有进位的位置的值赋给1，然后将其左移一位获得进位后的值；

重复上面两个步骤，直到进位的值为0中止。

```
class Solution {
public:
    int add(int num1, int num2){
        while (num2) {
            int sum = num1 ^ num2;
            int carry = num1 & num2;
            num1 = sum;
            num2 = carry << 1;
        }
        return num1;
    }
};
```

# 构建乘积数组

题目：https://www.acwing.com/problem/content/82/

题意：

给定一个数组`A[0, 1, …, n-1]`，请构建一个数组`B[0, 1, …, n-1]`，其中B中的元素`B[i]=A[0]×A[1]×… ×A[i-1]×A[i+1]×…×A[n-1]`。

不能使用除法。

思路：

首先从前到后，存储当前元素左半的乘积，再从后到前，存储右半乘积。从前到后可以直接利用f存储之前的值，从后到前可以定义一个临时变量存放右半乘积。

```
class Solution {
public:
    vector<int> multiply(const vector<int>& A) {
        int n = A.size();
        vector<int> f(n, 1);
        for (int i = 1; i < n; i ++) {
            f[i] = f[i - 1] * A[i - 1];
        }
        int t = 1;
        for (int i = n - 2; i >= 0; i --) {
            t *= A[i + 1];
            f[i] *= t;
        }
        return f;
    }
};
```

# 树中两个结点的最低公共祖先

题目：https://www.acwing.com/problem/content/84/

题意：

给出一个二叉树，输入两个树节点，求它们的最低公共祖先。

一个树节点的祖先节点包括它本身。

思路：

经典递归题。

返回条件是当前节点为空，或者等于所要找的p或q节点。

递归找左节点和右节点是否包含p或q节点，有以下两种情况：

1）左右节点都存在p或q，则表明当前节点就是最低公共祖先；

2）只有左节点或者只有右节点非空，则递归求得的非空的节点就是最低公共祖先；

```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (!root || root == p || root == q) return root;
        auto l = lowestCommonAncestor(root->left, p, q);
        auto r = lowestCommonAncestor(root->right, p, q);
        if (l && r) return root;
        if (l) return l;
        return r;
    }
};
```

# 把字符串转换成整数

题目：https://www.acwing.com/problem/content/83/

思路：

模拟题，注意边界

```
class Solution {
public:
    int strToInt(string str) {
        int res = 0, u = 0;
        bool isNeg = false;
        while (str[u] == ' ') u ++;
        if (str[u] == '+') u ++;
        if (str[u] == '-') {
            isNeg = true;
            u ++;
        }
        while (u < str.size()) {
            if (str[u] < '0' || str[u] > '9') break;
            int cur = str[u] - '0';
            if (res == INT_MAX / 10) {
                if (isNeg && cur >= abs(INT_MIN % 10)) return INT_MIN;
                if (!isNeg && cur >= INT_MAX % 10) return INT_MAX;
            } else if (res > INT_MAX / 10) {
                return isNeg ? INT_MIN : INT_MAX;
            }
            u ++;
            res = res * 10 + cur;
        }
        return isNeg ? -res : res;
    }
};
```



