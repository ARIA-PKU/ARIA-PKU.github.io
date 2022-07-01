---
title: DFS + 回溯
date: 2022-06-11T20:58:39+08:00
lastmod: 2022-06-11T20:58:39+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/stary_night.jpg

categories:
  - 算法题整理
tags:
  - DFS
# nolastmod: true
draft: false
---

DFS + 回溯的题型总结

<!--more-->

# [46. 全排列](https://leetcode.cn/problems/permutations/)

所有元素均不相同。

思路：

用一个state状态表示，然后dfs遍历并且回溯即可。

```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    void dfs(vector<int>& nums, int state) {
        if (path.size() == nums.size()) {
            ans.push_back(path);
            return;
        }

        for (uint32_t i = 0; i < nums.size(); i ++) 
            if (!(state >> i & 1)) {
                path.push_back(nums[i]);
                dfs(nums, state + (1 << i));
                path.pop_back();
            }
                
        
    }
    vector<vector<int>> permute(vector<int>& nums) {
        dfs(nums, 0);
        return ans;
    }
};
```

# [47. 全排列 II](https://leetcode.cn/problems/permutations-ii/)

可包含重复元素。

思路：

需要先将原来数组中的元素sort一遍，整理为有序。将数组中的所有元素插入到path数组中，为了保证不重复，如果当前元素与数组前一个元素相同（因为排序过，所以有相同的必在前面）则必须插入到相同元素的后面，保证相对有序，这样就不会产生重复结果。

```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    // st 记录前一个元素放的位置，u记录遍历到了有序数组的第几个,state记录path数组存放情况
    void dfs(vector<int>& nums, int st, int u, int state) {
        if (u == nums.size()) {
            ans.push_back(path);
            return;
        }
        // 不同则可以从头遍历
        if (!u || nums[u] != nums[u -1]) st = 0;
        for (uint32_t i = st; i < nums.size(); i ++) {
            if (!(state >> i & 1)) {
                path[i] = nums[u];
                dfs(nums, i, u + 1, state + (1 << i));
            }
        }
    }
    vector<vector<int>> permuteUnique(vector<int>& nums) {
        int n = nums.size();
        path.resize(n);
        sort(nums.begin(), nums.end());
        dfs(nums, 0, 0, 0);
        return ans;
    }
};
```

# [78. 子集](https://leetcode.cn/problems/subsets/)

所有元素互不相同，求所有子集。

思路：

直接暴力枚举所有状态，时间复杂度的为$O(2^n)$。

```
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        vector<vector<int>> ans;
        int n = nums.size();
        for (int i = 0; i < 1 << n; i ++) {
            vector<int> path;
            for (int j = 0; j < n; j ++) {
                if (i >> j & 1) path.push_back(nums[j]);
            }
            ans.push_back(path);
        }
        return ans;
    }
};
```

# [90. 子集 II](https://leetcode.cn/problems/subsets-ii/)

包含重复元素，枚举所有不同的子集

思路：

对原数组进行排序，然后dfs+回溯，每次枚举出下个不同元素的位置作为下次dfs起始位置，将当前元素个数从0枚举到原数组中个数，最后对其回溯。

```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    void dfs(vector<int>& nums, int u) {
        int n = nums.size();
        if (u == n) {
            ans.push_back(path);
            return;
        }
        // j统计当前元素个数，下一次dfs起始位置
        int j = u + 1;
        while (j < n && nums[j] == nums[u]) j ++;
        // 情况是从0个到j-u个
        for (int i = u; i <= j; i ++) {
            dfs(nums, j);
            path.push_back(nums[u]);
        }
        for (int i = u; i <= j; i ++)
            path.pop_back();
    }
    vector<vector<int>> subsetsWithDup(vector<int>& nums) {
        sort(nums.begin(), nums.end());
        dfs(nums, 0);
        return ans;
    }
};
```

# [39. 组合总和](https://leetcode.cn/problems/combination-sum/)

题意：

元素无重复，并且元素都是**正数**，不限制使用次数，求满足组合总数的所有集合。

思路：

和上一题子集Ⅱ基本一致，不过也有其他思路。

```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    void dfs(vector<int>& candidates, int st, int u) {
        if(u == 0) {
            ans.push_back(path);
            return;
        }
        // 越界则跳出
        if (st >= candidates.size()) return;
        // 找到最多次数j
        int j = u / candidates[st];
        for (int i = 0; i <= j; i ++) {
            dfs(candidates, st + 1, u - i * candidates[st]);
            path.push_back(candidates[st]);
        }
        for (int i = 0; i <= j; i ++) 
            path.pop_back();
    }
    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
        dfs(candidates, 0, target);
        return ans;
    }
};
```

# [40. 组合总和 II](https://leetcode.cn/problems/combination-sum-ii/)

题意：

包含重复元素，每个元素最多使用一次，元素全为正数，要求解集不重复。

思路：

思路也与上面的基本一致，具体细节的地方稍作修改。

```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    void dfs(vector<int>& candidates, int u, int target) {
        if (target == 0) {
            ans.push_back(path);
            return;
        }
        if (target < 0 || u >= candidates.size()) return;
        int j = u + 1;
        while (j < candidates.size() && candidates[j] == candidates[u]) j ++;
        for (int i = u; i <= j; i ++) {
            dfs(candidates, j, target - (i - u) * candidates[u]);
            path.push_back(candidates[u]);
        }
        for (int i = u; i <= j; i ++) path.pop_back();
    }
    vector<vector<int>> combinationSum2(vector<int>& candidates, int target) {
        sort(candidates.begin(), candidates.end());
        dfs(candidates, 0, target);
        return ans;
    }
};
```

# [216. 组合总和 III](https://leetcode.cn/problems/combination-sum-iii/)

题意：

和为n的k个数，只能从1到9中选数，最多选一个。

思路：

回溯+剪枝

```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> path;
    int k;
    void dfs(int cnt, int n, int st) {
        if (cnt > k || n < 0) return;
        if (n == 0 && cnt == k) {
            ans.push_back(path);
            return;
        }
        if (st >= 9) return;
        dfs(cnt, n, st + 1);
        path.push_back(st + 1);
        dfs(cnt + 1, n - st - 1, st + 1);
        path.pop_back();
    }
    vector<vector<int>> combinationSum3(int k, int n) {
        this->k = k;
        dfs(0, n, 0);
        return ans;
    }
};
```

# [22. 括号生成](https://leetcode.cn/problems/generate-parentheses/)

题意：

给定小于等于8的正整数n，求能生成的括号集合。

思路：

dfs，注意右括号个数不能多于左括号即可。

```
class Solution {
public:
    vector<string> ans;
    int n;
    void dfs(string u, int l, int r) {
        if (l == n && r == n) {
            ans.push_back(u);
            return;
        }
        if (l < n) dfs(u + '(', l + 1, r);
        if (r < n && r < l) dfs(u + ')', l, r + 1);
    }
    vector<string> generateParenthesis(int n) {
        this->n = n;
        string u = "";
        dfs(u, 0, 0);
        return ans;
    }
};
```

# [473. 火柴拼正方形](https://leetcode.cn/problems/matchsticks-to-square/)

题意：

给定若干火柴棒，不能折断，所有都要使用，判断能否拼成正方形。数组元素个数小于等于15。

思路：

经典问题的DFS剪枝问题，总结一下基本的剪枝策略。

枚举的时候，**按照每条边枚举**，枚举完每条边记录当前使用过的火柴。

1、枚举火柴的时候，从**大到小开始枚举**，因为开始枚举的越大，后面的选择越少，分支越少，越容易提前发现错误剪枝。

2、每条边内部，要求火柴**编号递增**

3、如果当前放某根火柴**失败**，

​	3.1、跳过长度相同的火柴

​	3.2 、如果是第一根火柴，则剪掉当前分支

​	3.3、如果是最后一根，也剪掉当前分支

```
class Solution {
public:
    int n, avg;
    vector<int> nums;
    vector<bool> used;
    // u记录当前长度，st记录使用到数组第几根火柴，cnt记录拼好的边数
    bool dfs(int u, int st, int cnt) {
        if (cnt == 3) return true;
        if (u == avg) return dfs(0, 0, cnt + 1);
        for (int i = st; i < n; i ++) {
            if (used[i]) continue;
            if (u + nums[i] <= avg) {
                used[i] = true;
                if (dfs(u + nums[i], i + 1, cnt)) return true;
                used[i] = false;
            }
            // 第一个个最后一个直接剪枝
            if (!u || u + nums[i] == avg) return false;
            // 相同长度火柴直接跳过
            while (i + 1 < nums.size() && nums[i + 1] == nums[i]) i ++;
        }
        return false;
    }
    bool makesquare(vector<int>& matchsticks) {
        n = matchsticks.size();
        used.resize(n);
        nums = matchsticks;
        avg = 0;
        for(auto& x: nums) avg += x;
        if (avg % 4) return false;
        avg /= 4;
        sort(nums.begin(), nums.end(), greater<int>());
        return dfs(0, 0, 0);
    }
};
```

# [52. N皇后 II](https://leetcode.cn/problems/n-queens-ii/)

题意：

求N皇后排列个数

思路：

dfs + 回溯，逐行遍历每行只有一个存放位置，则可以无需记录行的重复；需要额外的三个数组记录被占用的列号、主对角线和副对角线编号（由于主副对角线性质，两个数组即可）

```
const int N = 20;
bool col[N], dg[N], udg[N];
class Solution {
public:
    int ans;
    void dfs(int u, int n) {
        if (u == n) {
            ans ++;
            return;
        }
        for (int i = 0; i < n; i ++) {
            if (!col[i] && !dg[u + i] && !udg[n + u - i]) {
                // 主对角线是和为定值，副对角线是差为定值，+n为了避免为负数
                col[i] = dg[u + i] = udg[n + u - i] = true;
                dfs(u + 1, n);
                col[i] = dg[u + i] = udg[n + u - i] = false;
            }
        }
    }
    int totalNQueens(int n) {
        ans = 0;
        dfs(0, n);
        return ans;
    }
};
```

# [37. 解数独](https://leetcode.cn/problems/sudoku-solver/)

题意：

9*9的数独，有唯一解，求其唯一解

思路：

不同于N皇后问题，这个的枚举不能逐行枚举，需要逐个单元格枚举。

小技巧：可以定义为bool类型返回值，剪枝提前返回。

```
class Solution {
public:
    bool col[9][9], row[9][9], block[9][9];
    // bool类型为了剪枝提前返回
    bool dfs(vector<vector<char>>& board, int x, int y) {
        if (y == 9) x ++, y = 0;
        if (x == 9) return true;
        if (board[x][y] != '.') return dfs(board, x, y + 1);
        for (int i = 0; i < 9; i ++) {
            int k = (x / 3) * 3 + (y / 3);
            if (!row[x][i] && !col[y][i] && !block[k][i]) {
                row[x][i] = col[y][i] = block[k][i] = true;
                board[x][y] = '1' + i;
                if (dfs(board, x, y + 1)) return true;
                board[x][y] = '.';
                row[x][i] = col[y][i] = block[k][i] = false;
            }
        }
        return false;
    }
    void solveSudoku(vector<vector<char>>& board) {
        memset(row, false, sizeof row);
        memset(col, false, sizeof col);
        memset(block, false, sizeof block);
        for (uint32_t i = 0; i < board.size(); i ++) {
            for (uint32_t j = 0; j < board[0].size(); j ++) {
                if (board[i][j] != '.') {
                    int u = board[i][j] - '1';
                    int k = (i / 3) * 3 + (j / 3);
                    row[i][u] = col[j][u] = block[k][u] = true;
                }
            }
        }
        dfs(board, 0, 0);
        return;
    }
};
```

# [282. 给表达式添加运算符](https://leetcode.cn/problems/expression-add-operators/)

题意：

给字符串添加符号，使得其计算结果等于给定的target，添加的符号限定为$+, -、*$，求所有能生成的集合。

思路：

特别复杂的一道题目。

基本思路是暴力枚举所有表达式方案:

每两个数字之间可以不填任何字符，也可以填加减乘，所以每个间隔有4种方案，直接暴力枚举所有方案即可。

本题的难点在于常数优化，如果实现方式不好，那么本题就很容易超时。总结下来一共两点：

1、如果先暴搜出所有表达式的形式，然后再用表达式求值的模板去求解，那么会超时；

2、记录方案时如果每次复制整个数组，那么会超时；

为了能尽量优化常数，我们在递归过程中尽量不要使用栈来维护表达式。本题中我们维护如下不变式：

$a+b×()$，其中括号中的数是我们枚举的下一个数；然后分类讨论下一个运算符，其中 c是我们枚举的括号中的数：

下一个运算符是加号，那么 $a+b×(c)+()=(a+b×c)+1×();$

下一个运算符是减号，那么 $a+b×(c)−()=(a+b×c)+(−1)×();$

下一个运算符是乘号，那么 $a+b×(c)×()=a+(b×c)×();$

为了方便，我们在表达式最后统一添加一个加号，那么最终不变式就会变成 $a+1×()$，所以 a 就是我们枚举的表达式的值，判断一下和target是否相等即可。

DFS枚举，**时间复杂度**为$O(n*4^n)$

```
typedef long long LL;
class Solution {
public:
    vector<string> ans;
    string path;
    // u是在num中的当前位置，len是path当前长度
    void dfs(string& num, int u, int len, LL a, LL b, int target) {
        if (u == num.size()) {
            if (a == target) ans.push_back(path.substr(0, len - 1));
        } else {
            LL c = 0;
            for (uint32_t i = u; i < num.size(); i ++) {
                c = c * 10 + num[i] - '0';
                path[len ++] = num[i];

                // 当前位置添加+
                path[len] = '+';
                dfs(num, i + 1, len + 1, a + b * c, 1, target);

                if (i + 1 < num.size()) {
                    // -
                    path[len] = '-';
                    dfs(num, i + 1, len + 1, a + b * c, -1, target);

                    // *
                    path[len] = '*';
                    dfs(num, i + 1, len + 1, a, b * c, target);
                }
                // 第一次循环是作为0计算，后面则作为前导零需要break掉
                if (num[u] == '0') break;
            }
        }
    }
    vector<string> addOperators(string num, int target) {
        path.resize(100);
        dfs(num, 0, 0, 0, 1, target);
        return ans;
    }
};
```

# [301. 删除无效的括号](https://leetcode.cn/problems/remove-invalid-parentheses/)

题意：

给你一个由若干括号和字母组成的字符串 `s` ，删除最小数量的无效括号，使得输入的字符串有效。

返回所有可能的结果。答案可以按 **任意顺序** 返回。

思路：

剪枝思路：

1、先预处理一遍原数组，找到原字符串中最多删除的左右括号作为dfs的剪枝条件

2、在处理连续的左括号和右括号的时候，采用逐个添加再dfs的方式，这个方法很多回溯的方法中都用过，可以避免重复

另外注意除了左右括号还有其他字母，处理的时候需要注意。

```
class Solution {
public:
    vector<string> ans;
    // u代表枚举到s的第几位，path记录当前结果
    // cnt记录当前结果左括号与右括号差值，l和r分别代表当前最多还可以删除的对应括号数
    void dfs(string& s, int u, string path, int cnt, int l, int r) {
        if (u == s.size()) {
            if (cnt == 0) ans.push_back(path);
            return;
        }
        if (s[u] == '(') {
            int k = u;
            while (k < s.size() && s[k] == '(') k ++;
            l -= k - u;
            // 下面是依次添加括号，添加从0到k-u个括号
            // 注意下一次枚举的起始位置是k
            for (int i = 0; i <= k - u; i ++) {
                if (l >= 0) dfs(s, k, path, cnt, l, r);
                path += '(';
                l ++, cnt ++;
            }
        } else if (s[u] == ')') {
            int k = u;
            while (k < s.size() && s[k] == ')') k ++;
            r -= k - u;
            for (int i = 0; i <= k - u; i ++) {
                if (cnt >= 0 && r >= 0) dfs(s, k, path, cnt, l, r);
                path += ')';
                r ++, cnt --;
            }
        } else {
            dfs(s, u + 1, path + s[u], cnt, l, r);
        }
    }
    vector<string> removeInvalidParentheses(string s) {
        int l = 0, r = 0;
        for (auto& x: s) {
            if (x == '(') l ++;
            else if (x == ')') {
                if (l > 0) l --;
                else r ++;
            }
        }
        dfs(s, 0, "", 0, l, r);
        return ans;
    }
};
```

