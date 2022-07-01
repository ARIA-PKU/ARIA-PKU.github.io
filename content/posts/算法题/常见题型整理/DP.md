---
title: 动态规划
date: 2022-06-08T22:55:24+08:00
lastmod: 2022-06-08T22:55:24+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/stary_night.jpg
# images:
#   - /img/cover.jpg
categories:
  - 算法题整理
tags:
  - 动态规划
# nolastmod: true
draft: false
---

总结一下遇到的动态规划题型，都是做过几遍的问题，再重新整理一遍。

<!--more-->

根据y总的总结，所有DP问题都可以分为**状态表示**和**状态计算**两个部分，其中状态表示可以细分为**集合划分**和**属性确定**，属性一般分为最大、最小值等，集合划分与状态计算则依问题不同而不同，但是大多数问题都有解决的共同方法，接下来的实际问题中会总结分析相应方法。

# 一、背包问题（基础版）

## 01背包

**题目：**https://www.acwing.com/problem/content/2/

**思路：**

1、状态表示：`f[i, j]`

​	集合划分：所有只考虑前i个物品，且总体积不超过j的集合

​	属性：满足条件的总价值最大的背包

2、状态计算：

​	对于任意状态`f[i, j]`，其对于第`i`个物品有两种选择方法，不选第`i`件物品对应的方案等价于`f[i - 1, j]`，选择第`i`件对应的状态为`f[i - 1, j - v[i]] + w[i]`，由于取最大值，所以两者取其大即可。

根据上面的分析，可以得到状态转移方程：

```
f[i , j] = max(f[i - 1, j], f[i - 1, j - v[i]] + w[i])
```

01背包的常用解法还进行了空间优化，观察到`i`状态只依赖于`i - 1`的状态，在`j`的循环中，从大到小循环，每次的`f[j]`更新使用的都是`j`更小的上一次循环的内容，因此可以直接将空间压缩到一维。

```
#include <iostream>
using namespace std;

const int N = 1010;
int w[N], v[N], f[N];
int n, s;

int main() {
    scanf("%d%d", &n, &s);
    for (int i = 0; i < n; i ++) scanf("%d%d", &w[i], &v[i]);
    for (int i = 0; i < n; i ++) {
        for (int j = s; j >= w[i]; j --) {
            f[j] = max(f[j], f[j - w[i]] + v[i]);
        }
    }
    printf("%d", f[s]);
    return 0;
}
```

## 完全背包

**题目：**https://www.acwing.com/problem/content/3/

**思路：**

完全背包在代码上和01背包可以说是及其相似，但是思路差别挺大的，写的时候可以不用想那么多，总结的时候还是仔细分析一下。

1、状态表示：`f[i, j]`

​	集合划分：所有只考虑前`i`个物品，且总体积不超过`j`的集合

​	属性：满足条件的总价值最大的背包

2、状态计算：

​	对于任意状态`f[i, j]`，其对于第`i`个物品很多种选择方法，不选第`i`件物品对应的方案等价于`f[i - 1, j]`，第`i`件选择`k`件对应的状态为$f[i - 1, j - k * v[i]] + k * w[i]$，由于取最大值，对所有状态取最大的即可。

推导如下：

```
f[i, j] = max(f[i - 1, j], f[i - 1, j - v] + w, ... , f[i - 1, j - k * v] + k * w, ...)

f[i, j - v] = max(f[i - 1, j - v], f[i - 1, j - 2v] + w, ..., f[i - 1, j - k * v] + (k - 1)w, ...)
```

观察两个式子很容易发现规律：

```
f[i, j] = max(f[i - 1, j], f[i, j - v] + w)
```

接下来，类似01背包进行空间优化，但是这里用来更新`j`的是本次`i`状态下的值，而不是`i - 1`状态下的值，因此`j`需要从小到大更新:

```
#include <iostream>
using namespace std;

const int N = 1010;
int v[N], w[N], f[N];
int n, s;

int main() {
    scanf("%d%d", &n, &s);
    for (int i = 0; i < n; i ++) scanf("%d%d", &w[i], &v[i]);
    for (int i = 0; i < n; i ++) {
        for (int j = w[i]; j <= s; j ++) {
            f[j] = max(f[j], f[j - w[i]] + v[i]);
        }
    }
    printf("%d", f[s]);
    return 0;
}
```

## 多重背包问题

**题目1（基础版）**：https://www.acwing.com/problem/content/4/

**题目2（二进制优化版）**：https://www.acwing.com/problem/content/5/

**思路：**

多重背包问题在数据量较小的情况下，可以考虑三重循环解决，在数据量较大的时候需要考虑进行二进制优化。

1、状态表示：`f[i, j]`

​	集合划分：所有只考虑前`i`个物品，且总体积不超过`j`的集合，对应物品选择不超过`s`件

​	属性：满足条件的总价值最大的背包

2、状态计算：

​	类似前面的分析，其中`s`为第`i`件物品的个数。

```
f[i, j] = max(f[i - 1, j], f[i - 1, j - v] + w, ... , f[i - 1, j - s * v] + s * w)
```

更新用到的是前一个轮次`j`更小的值，因此`j`的循环从大到小进行，可以得到基础版解决方法，时间复杂度为$O(N*M*S)$：

```
#include <iostream>
using namespace std;

const int N = 110;
int w[N], v[N], s[N], f[N];
int n, m;

int main() {
    scanf("%d%d", &n, &m);
    for (int i = 0; i < n; i ++) scanf("%d%d%d", &w[i], &v[i], &s[i]);
    for (int i = 0; i < n; i ++)
        for (int j = m; j >= 0; j --)
            for (int k = 0; k * w[i] <= j && k <= s[i]; k ++) 
                f[j] = max(f[j], f[j - k * w[i]] + k * v[i]);
    
    printf("%d\n", f[m]);
    return 0;
}
```

在数据量大的时候，会TLE，通过简单公式推导可知不能用类似完全背包的方法（因为会多出来一项无法处理）。

**二进制优化思路：**

假设我们的最大可选个数为S等于1023，我们可选的方案有1， 2， 3， .....，1023，如果我们依次遍历则总共要1023次；

可以用二进制的思路分解为$2^0, 2^1, 2^2, ......, 2^9$，可以利用这10个数组合出从1到1023的所有数字，通过选与不选的方式将原来的问题转换为了01背包的问题。时间复杂度降低到了$O(N*M*logS)$

```
#include <iostream>
using namespace std;

const int N = 11010, M = 2010;
int w[N], v[N], f[M];

int main() {
    int n, m;
    scanf("%d%d", &n, &m);
    int a, b, s, cnt = 0;
    for (int i = 0; i < n; i ++) {
        scanf("%d%d%d", &a, &b, &s);
        int k = 1;
        // 二进制处理
        while (k < s) {
            w[cnt] = k * a;
            v[cnt] = k * b;
            s -= k;
            k *= 2;
            cnt ++;
        }
        if (s) { // 将剩余的加进来
            w[cnt] = s * a;
            v[cnt] = s * b;
            cnt ++;
        }
    }
    // 转换为了01背包问题
    for (int i = 0; i < cnt; i ++)
        for (int j = m; j >= w[i]; j --)
            f[j] = max(f[j], f[j - w[i]] + v[i]);
    
    printf("%d\n", f[m]);
    return 0;
}
```

## 分组背包问题

**题目：**https://www.acwing.com/problem/content/9/

**思路：**

1、状态表示：`f[i, j]`

​	集合划分：所有只考虑前`i`组物品，且总体积不超过`j`的集合

​	属性：满足条件的总价值最大的背包

2、状态计算：

​	类似前面的分析，对第`i`组而言，需要找到以下条件的：

```
for (int i = 0; i < n; i ++)
{
	for (int j = m; j >= v; j --)
		// 在第j组内取使得总价值最大的
		f[j] = max(f[j], f[j-v[0]] + w[0], f[j-v[1]]+w[1],...,f[j - v[s - 1]] + w[s - 1]);
}
```

因此，需要三重循环处理问题。背包问题在优化完空间之后，循环顺序必须是物品、体积、决策。

```
#include <iostream>
using namespace std;

const int N = 110;
int w[N], v[N], f[N];

int main() {
    int n, m, s;
    scanf("%d%d", &n, &m);
    for (int i = 0; i < n; i ++) { // 物品
        scanf("%d", &s);
        for (int j = 0; j < s; j ++) scanf("%d%d", &w[j], &v[j]);
        for (int j = m; j >= 0; j --) // 体积
            for (int k = 0; k < s; k ++)  // 决策，在第i组中不选或者选择哪一个
                if (w[k] <= j) f[j] = max(f[j], f[j - w[k]] + v[k]);
    }
    printf("%d\n", f[m]);
    return 0;
}
```

# 二、线性DP

## 最长上升子序列

典中典线性DP问题，各大平台必不可少的题目。

题目1（基础dp版）：https://www.acwing.com/problem/content/897/

题目2（进阶dp版）：https://www.acwing.com/problem/content/898/

题目3（lc版）：https://leetcode.cn/problems/longest-increasing-subsequence/

题目4（NC91），在上面的基础上增加了输出最长序列的要求

**思路：**

基础版的两重循环，简单易懂：

```
class Solution {
public:
    int lengthOfLIS(vector<int>& nums) {
        int res = 1;
        uint32_t n = nums.size();
        vector<int> f(n + 1, 1);
        for(uint32_t i = 0; i < nums.size(); i ++) {
            for (uint32_t j = 0; j < i; j ++) {
                if (nums[i] > nums[j]) f[i] = max(f[i], f[j] + 1);
                res = max(res, f[i]);
            }
        }
        return res;
    }
};
```

维护一个单调递增的序列，用后面的元素更新单调序列，通过二分法找到当前元素在单调序列中的位置并更新，通过p数组记录其在单调序列中的位置，然后逆序输出即可。

```
const int N = 3000;
int f[N], p[N];  // p是为了输出子序列
class Solution {
public:
    int lengthOfLIS(vector<int>& nums) {
        memset(f, 0, sizeof f);
        int len = 0;
        for (uint32_t i = 0; i < nums.size(); i ++) {
            int l = 0, r = len;
            // 找到小于当前值的最大值的位置
            while (l < r) {
                int mid = l + r + 1 >> 1;
                if (f[mid] < nums[i]) l = mid;
                else r = mid - 1;
            }
            // 将当前值更新到队列中第一个大于等于当前值的位置
            f[r + 1] = nums[i];
            p[i] = r + 1;
            len = max(len, r + 1);
        }
        // 以下是输出序列的处理
        vector<int> ans;
        int l = len;
        uint32_t n = nums.size();
        for (int i = n - 1; i >= 0; i --) {
        // 只会输出之前的第一个符合长度条件的值，后面更新的位置不会被输出
            if (p[i] == l) {  
                ans.push_back(nums[i]);
                l --;
            }
        }
        reverse(ans.begin(), ans.end());
        return len;
    }
};
```

## 编辑距离

这也是经典的线性DP问题，变种也非常多，但是基本思路是一致的。

**题目1（lv1）：**https://leetcode.cn/problems/edit-distance/

**题目2（lv2）：**NC35，将插入、删除和修改的代价赋值

**题目3（lv3）：**没找到题目，意思是从A修改成B的修改过程输出

**思路：**

本质上，从A修改成B和将A和B同时修改是一致的。下面会举例说明。

1、状态表示：`f[i, j]`

​	集合划分：表示A的前i个元素和B的前j个元素相同时所需修改的次数

​	属性：满足条件的最小修改次数

2、状态计算：

​	分析前一步的差异，总共可以分为六种情况。

1）删除A的最后一个元素。这个则说明A的前i - 1个元素已经和B的前j个元素在$f[i - 1][j]$步内相同，即$f[i][j] = f[i - 1][j] + 1$

2)  在A最后添加元素。此时添加的元素必定为$B[j]$，即在加之前，已经有A的1到i等于B的1到j - 1，可得$f[i][j] = f[i][j - 1] + 1$

3)  修改A的最后一个元素，根据A的最后一个元素与B的最后一个是否相等，可以得到$f[i][j] = f[i - 1][j - 1] + (A[i] != B[j])$

4)  删除B的最后一个元素，这个对应了前面的情况2）

5）在B最后添加一个元素，对应情况1)

6) 修改B的最后一个元素，对应情况3）

综上可知，实际上只有3种情况需要考虑，对应存在不同代价的，只要将1修改成相应代价即可。

```
class Solution {
public:
    int minDistance(string word1, string word2) {
        int m = word1.size(), n = word2.size();
        word1 = " " + word1, word2 = " " + word2;
        vector<vector<int>> f(m + 1, vector<int>(n + 1));
        for (int i = 0; i <= m; i ++) f[i][0] = i;
        for (int i = 1; i <= n; i ++) f[0][i] = i;
        for (int i = 1; i <= m; i ++)
            for (int j = 1; j <= n; j ++) {
                f[i][j] = min(f[i - 1][j], f[i][j - 1]) + 1;
                f[i][j] = min(f[i][j], f[i - 1][j - 1] + (word1[i] != word2[j]));
            }
        return f[m][n];
    }
};
```

输出修改路径，维护一个path路径，利用A、D、R、N分别记录增删改和不变。最终逆序输出即可。

```
class Solution {
public:
    /**
     * min edit cost
     * @param str1 string字符串 the string
     * @param str2 string字符串 the string
     * @param ic int整型 insert cost
     * @param dc int整型 delete cost
     * @param rc int整型 replace cost
     * @return int整型
     */
    int minEditCost(string s1, string s2, int ic, int dc, int rc) {
        // write code here
        int m = s1.size(), n = s2.size();
        vector<vector<int>> f(m + 1, vector<int>(n + 1));
        vector<vector<char>> path(m + 1, vector<char>(n + 1));
        s1 = " " + s1, s2 = " " + s2;
        for (int i = 0; i <= m; i ++) {
            f[i][0] = i * dc;
            path[i][0] = 'D';
        }
        for (int i = 0; i <= n; i ++) {
            f[0][i] = i * ic;
            path[0][i] = 'A';
        }
        for (int i = 1; i <= m; i ++) {
            for (int j = 1; j <= n; j ++) {
                if (f[i - 1][j] + dc > f[i][j - 1] + ic) {
                    f[i][j] = f[i][j - 1] + ic;
                    path[i][j] = 'A';
                }
                else {
                    f[i][j] = f[i - 1][j] + dc;
                    path[i][j] = 'D';
                }
				// 相同则无需修改
                if (s1[i] == s2[j]) {
                    if (f[i][j] > f[i - 1][j - 1]) {
                        f[i][j] = f[i - 1][j - 1];
                        path[i][j] = 'N';
                    }
                }
                else {
                    if (f[i][j] > f[i - 1][j - 1] + rc) {
                        f[i][j] = f[i - 1][j - 1] + rc;
                        path[i][j] = 'R';
                    }
                }
            }
        }
        // 逆序输出修改方式
        int l = m, r = n;
        while (l != 0 && r != 0) {
            if (path[l][r] == 'R') {
                cout << "R " << s1[l] << " " << s2[r] << endl;
                l --, r --;
            }
            else if (path[l][r] == 'N') {
                l --, r --;
            }
            else if (path[l][r] == 'A') {
                cout << "A " << s2[r] << endl;
                r --;
            }
            else {
                cout << "D " << s1[l] << endl;
                l --;
            }
        }
        return f[m][n];
    }
};
```

# 三、股票问题系列

这一类问题好多，有的问题都不是DP，有的是状态机DP，统一整理起来。

## [121. 买卖股票的最佳时机](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/)

只交易一次

思路：简单的贪心

```
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int res = 0, u = 10010;
        for (auto& x: prices) {
            if (x < u) u = x;
            res = max(res, x - u);
        }
        return res;
    }
};
```

## [122. 买卖股票的最佳时机 II](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-ii/)

交易无限次

思路：也是简单的贪心

```
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int n = prices.size(), res = 0;
        for (int i = 1; i < n; i ++) {
            if (prices[i] > u) res += prices[i] - u;  
        }
        return res;
    }
};
```

## [123. 买卖股票的最佳时机 III](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iii/)

交易两次

思路：

由于只买卖两次，所以有一种特殊的思路，将整个买卖分成前后两部分，先从前往后获得一次买卖的最大收益，再从后向前，确定当前买卖的最大收益并与之前计算得到的收益累加比较求得最大值。

```
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int n = prices.size();
        vector<int> f(n);
        int u = prices[0];
        for (int i = 1; i < n; i ++) {
            f[i] = f[i - 1];
            if (prices[i] > u) f[i] = max(f[i], prices[i] - u);
            u = min(u, prices[i]);
        }
        u = prices[n - 1];
        int res = 0, pre = 0;
        for (int i = n - 2; i >= 0; i --) {
            if (prices[i] < u) pre = max(pre, u - prices[i]);
            u = max(u, prices[i]);
            res = max(res, pre + f[i]);
        }
        return res;
    }
};
```

## [188. 买卖股票的最佳时机 IV](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iv/)

交易k次

思路：

状态机DP，感觉有点难。

首先是特判k的次数，超过n/2次就相当于次数无限制了；

然后，分析状态机，四种转移方式：

1）不持有，不交易  +0

2）不持有，买入 -w[i]

3）持有，不交易 +0

4）持有，卖出 +w[i]

状态表示定义：

$f[i][j]$，表示经过i天，转了j圈的最大收益，即处于不持有状态最大收益

$g[i][j]$，表示经过i天，正在转j圈，即处于持有状态的最大收益

根据状态转移方式可得：

$f[i][j] = max(f[i - 1][j], g[i - 1][j] + w[i])$

$g[i][j] = max(g[i - 1][j], f[i - 1][j - 1] - w[i])$，这里是j-1，因为是从转了j-1圈的状态转移过来的。

```
class Solution {
public:
    int maxProfit(int k, vector<int>& prices) {
        int n = prices.size(), res = 0;
        if (k >= n / 2) {
            for (int i = 1; i < n; i ++)
                if (prices[i] > prices[i - 1])
                    res += prices[i] - prices[i - 1];
            return res;
        }

        vector<vector<int>> f(n + 1, vector<int>(k + 1, -1e8));
        auto g = f;
        f[0][0] = 0;
        for (int i = 1; i <= n; i ++)
            for (int j = 0; j <= k; j ++) {
                f[i][j] = max(f[i - 1][j], g[i - 1][j] + prices[i - 1]);
                g[i][j] = g[i - 1][j];
                if (j) g[i][j] = max(g[i][j], f[i - 1][j - 1] - prices[i - 1]);
                res = max(res, f[i][j]);
            }
        return res;
    }
};
```

空间优化小技巧，直接`&1`：

```
class Solution {
public:
    int maxProfit(int k, vector<int>& prices) {
        int n = prices.size(), res = 0;
        if (k >= n / 2) {
            for (int i = 1; i < n; i ++)
                if (prices[i] > prices[i - 1])
                    res += prices[i] - prices[i - 1];
            return res;
        }

        vector<vector<int>> f(2, vector<int>(k + 1, -1e8));
        auto g = f;
        f[0][0] = 0;
        for (int i = 1; i <= n; i ++)
            for (int j = 0; j <= k; j ++) {
                f[i & 1][j] = max(f[i - 1 & 1][j], g[i - 1 & 1][j] + prices[i - 1]);
                g[i & 1][j] = g[i - 1 & 1][j];
                if (j) g[i & 1][j] = max(g[i & 1][j], f[i - 1 & 1][j - 1] - prices[i - 1]);
                res = max(res, f[i & 1][j]);
            }
        return res;
    }
};
```

# 四、记忆化搜索

DFS + 记忆化搜索的类型，也算是DP吧（？）

## [329. 矩阵中的最长递增路径](https://leetcode.cn/problems/longest-increasing-path-in-a-matrix/)

相同题目：https://www.acwing.com/problem/content/903/

**思路：**

记忆化搜索的思路就是把搜索过程中的结果保存下来，避免重复计算最终导致TLE，一般与DFS搭配。

```
class Solution {
public:
    int dx[4] = {0, 1, 0, -1}, dy[4] = {1, 0, -1, 0};
    // g数组保存中间结果
    vector<vector<int>> g;
    int res, m, n;
    int f(vector<vector<int>>& matrix, int x, int y) {
        auto& v = g[x][y];
        if (g[x][y]) return v;
        v = 1;
        for (int i = 0; i < 4; i ++) {
            int nx = x + dx[i], ny = y + dy[i];
            if (nx >= 0 & nx < m && ny >= 0 && ny < n && matrix[nx][ny] < matrix[x][y]) {
                v = max(v, f(matrix, nx, ny) + 1);
            }
        }
        return v;
    }
    int longestIncreasingPath(vector<vector<int>>& matrix) {
        m = matrix.size(), n = matrix[0].size();
        g.resize(m, vector<int>(n));
        for (int i = 0; i < m; i ++) 
            for (int j = 0; j < n; j ++) 
                res = max(res, f(matrix, i, j));

        return res;
    }
};
```

# 五、状态压缩+DP

状态压缩，在N小于等于30的情况下可以考虑使用（当然该数据范围下也可以考虑dfs+剪枝）

## [526. 优美的排列](https://leetcode.cn/problems/beautiful-arrangement/)

**思路：**

状态表示：$f[i][state]$，表示填充了前i位，使用的数字为state状态的排列个数。这里的state状态就是数字的二进制表示法，0表示未使用，1表示使用。

状态转移：$f[i][state] += f[i - 1][state_s]$,其中state<sub>s</sub>表示的是所有满足要求的状态。

优化：

考虑到i的个数其实就等于state中1的个数，而且state必然是大于state<sub>s</sub>的，因此只要从大到小枚举即可。时间复杂度为$O(n*2^n)$，共有$2^n$种状态，n种决策数，转移时间为$O(1)$.

```
class Solution {
public:
    int countArrangement(int n) {
        vector<int> f(1 << n);
        f[0] = 1;
        // i 从小到大枚举状态
        for (int i = 0; i < 1 << n; i ++) {
            int k = 0; // k是记录了当前前k个位置已经被填充
            for (int j = 0; j < n; j ++) 
                if (i >> j & 1)
                    k ++;
			
			// state记录的是1到n中数字的使用情况，j枚举的是数字j+1是否被使用
            for (int j = 0; j < n; j ++) {
                if (!(i >> j & 1))
                	// 在状态中的第j位的值为j+1（因为从0开始计算状态），同样的第k个位置的下标为k+1
                    if ((j + 1) % (k + 1) == 0 || (k + 1) % (j + 1) == 0)
                        f[i | 1 << j] += f[i];
            } 
        }
        return f[(1 << n) - 1];
    }
};
```

