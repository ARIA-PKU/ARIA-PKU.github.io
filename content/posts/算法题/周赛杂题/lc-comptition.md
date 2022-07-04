---
title: leetcode的算法周赛题整理
date: 2022-06-12T14:55:05+08:00
lastmod: 2022-06-12T14:55:05+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/butterfly.jpg
# images:
#   - /img/cover.jpg
categories:
  - 周赛算法题
tags:
  - leetcode
  - 算法题
# nolastmod: true
draft: false
---

记录每周周赛题，滴水穿石

<!--more-->

## 294场周赛

### [2281. 巫师的总力量和](https://leetcode.cn/problems/sum-of-total-strength-of-wizards/)

题意为，一个子数组的力量为子数组最小值乘数组元素个数，求所有子数组的力量总和。

思路：

单调栈的套路常见，前缀和的找规律不熟，而且要注意边界细节

通过单调栈求出每个位置上，左侧第一个**小于** 当前力量值的 后一个 位置 li 以及右侧 **小于等于** 当前力量值的 前一个 位置 ri。
对于 i，定义闭区间 $[l_i,r_i]$ 为当前巫师的「管辖区间」，即包含以 ii 个巫师为最弱能力值的所有区间。
所有位置都会有一个管辖区间，每个位置的管辖区间对整体答案的贡献可以通过二级前缀和在常数时间内计算出来。
假设有了普通前缀和数组 sum，则第 i 个巫师的管辖区间可以拆分成如下区间和的和，$[l,r],[l+1,r],…,[i,r],[l,r−1],[l+1,r−1],…,[i,r−1],…,…,[i−1,i],[i,i]$，将以上区间写成前缀和相减的形式，可以发现，$sum_r,sum_r−1,…,sumi$ 出现了 i−l+1 次，$sumr,sumr−1,…,sumi$ 出现了 r−i+1 次。通过预处理前缀和的前缀和数组，可以快速求出以上的所有区间的和。

**时间复杂度**
单调栈预处理 l 和 r 数组仅需要线性的时间。
预处理后仅需要遍历数组一次，故总时间复杂度为 O(n)。

**空间复杂度**
需要 O(n)的额外空间存储前缀和数组和栈。

**C++ 代码**

```
#define LL long long

const int mod = 1000000007;

class Solution {
private:
    int calc(int l, int r, const vector<int> &s) {
        if (r < 0) return 0;
        if (l <= 0) return s[r];

        return (s[r] - s[l - 1]) % mod;
    }

public:
    int totalStrength(vector<int>& strength) {
        const int n = strength.size();

        vector<int> sum(n), s(n);
        sum[0] = strength[0];
        s[0] = sum[0];

        for (int i = 1; i < n; i++) {
            sum[i] = (sum[i - 1] + strength[i]) % mod;
            s[i] = (s[i - 1] + sum[i]) % mod;
        }

        stack<int> st;
        vector<int> l(n), r(n);

        strength.push_back(0);
        for (int i = 0; i <= n; i++) {
            while (!st.empty() && strength[i] <= strength[st.top()]) {
                int top = st.top();
                st.pop();

                r[top] = i - 1;
                if (st.empty()) l[top] = 0;
                else l[top] = st.top() + 1;
            }

            st.push(i);
        }

        int ans = 0;
        for (int i = 0; i < n; i++) {
            int w = strength[i];

            LL rsum = calc(i, r[i], s), lcnt = i - l[i] + 1;
            LL lsum = calc(l[i] - 1, i - 1, s), rcnt = r[i] - i + 1;

            ans = (ans + rsum * lcnt % mod * w) % mod;
            ans = (ans - lsum * rcnt % mod * w) % mod;
        }

        return (ans + mod) % mod;
    }
};
```

## 80场双周赛

### [6098. 统计得分小于 K 的子数组数目](https://leetcode.cn/problems/count-subarrays-with-score-less-than-k/)

思路：

思路并不复杂的题目，因为全是正数，所以具有单调性，可以考虑用二分和滑动数组的思路解决。

二分的思路时间复杂度$O(nlogn)$，而且二分起来也要考虑一下。

```
typedef long long LL;
class Solution {
public:
    long long countSubarrays(vector<int>& nums, long long k) {
        int n = nums.size();
        vector<LL> f(n + 1);
        for (int i = 1; i <= n; i ++) f[i] = f[i - 1] + nums[i - 1];
        LL res = 0;
        for (int i = 1; i <= n; i ++) {
        	// 对每个i，找到第一个满足结果大于等于k的值（因为是大于等于所以下面的r要是n + 1）
            int l = i, r = n + 1;
            while (l < r) {
                int mid = l + r >> 1;
                LL u = (LL)(f[mid] - f[i - 1]) * (mid - i + 1);
                if (u >= k) r = mid;
                else l = mid + 1;
            }
            // 这里要r-1，因为r是大于等于的
            res += r - i;
        }
        return res;
    }
};
```

滑动数组的思路更简单，时间复杂度也只有$O(N)$

```
typedef long long LL;
class Solution {
public:
    long long countSubarrays(vector<int>& nums, long long k) {
        int n = nums.size();
        vector<LL> f(n + 1);
        for (int i = 1; i <= n; i ++) f[i] = f[i - 1] + nums[i - 1];
        LL res = 0;
        for (int i = 1, j = 0; i <= n; i ++) {
            while ((f[i] - f[j]) * (i - j) >= k) j ++;
            res += i - j;
        }
        return res;
    }
};
```

## 297场周赛

### [5289. 公平分发饼干](https://leetcode.cn/problems/fair-distribution-of-cookies/)

题意大致是最均匀地将所有饼干分配成k份。

思路：

根据数据量来看，DFS或者状态压缩DP，可以采用DFS解决

```
class Solution {
public:
    int res = 1e8;
    int sum[8] = {0};
    
    void dfs(vector<int>& cookies, int u, int k) {
        if (u == cookies.size()) {
            int t = 0;
            for (int i = 0; i < k; i ++) t = max(t, sum[i]);
            res = min(res, t);
            return; 
        }

        for (int i = 0; i < k; i ++) {
            sum[i] += cookies[u];
            if (sum[i] < res) {
                dfs(cookies, u + 1, k);
            }
            sum[i] -= cookies[u];
        }
    }

    int distributeCookies(vector<int>& cookies, int k) {
        dfs(cookies, 0, k);
        return res;    
    }
};
```

## 298场周赛

### [5254. 卖木头块](https://leetcode.cn/problems/selling-pieces-of-wood/)

题意：

给定木板，和一组模板宽高和价值，求价值最大切法。

思路：

DP，一共两种切分方式，横着切和竖着切。对每种切法，枚举切成两块木板的价值总和。

```
typedef long long LL;
class Solution {
public:
    long long sellingWood(int n, int m, vector<vector<int>>& prices) {
        vector<vector<LL>> f(n + 1, vector<LL>(m + 1));
        // 初始化各个板子的价值
        for (auto& p: prices) {
            auto h = p[0], w = p[1], v = p[2];
            f[h][w] = v;
        }
        
        // 枚举长和宽
        for (int i = 1; i <= n; i ++) {
            for (int j = 1; j <= m; j ++) {
                // 横着切
                for (int k = 1; k < i; k ++) {
                    f[i][j] = max(f[i][j], f[k][j] + f[i - k][j]);
                }
                // 竖着切
                for (int k = 1; k < j; k ++) {
                    f[i][j] = max(f[i][j], f[i][k] + f[i][j - k]);
                }
            }
        }
        
        return f[n][m];
    }
};
```

## 299场周赛

### [6103. 从树中删除边的最小分数](https://leetcode.cn/problems/minimum-score-after-removals-on-a-tree/)

**题意：**

删除树中两条 不同 的边以形成三个连通组件。对于一种删除边方案，定义如下步骤以计算其分数：

分别获取三个组件 每个 组件中所有节点值的异或值。
最大 异或值和 最小 异或值的 差值 就是这一种删除边方案的分数。

**思路：**

第一次接触的思路：先确定第一个断开的连线再建树。

在建树之后，枚举两棵树中的各个线，再遍历，dfs两次，O(n^2)时间复杂度。

```
const int N = 1010, M = 2 * N;
int h[N], e[M], ne[M], idx;

class Solution {
public:
    int ans = 1e9;
    vector<int> w;
    void add(int a, int b) {
        e[idx] = b, ne[idx] = h[a], h[a] = idx ++;
    }
    // u是当前节点，fa是父节点，sumx是当前树的异或和，sumy是另一棵树的异或和
    int dfs(int u, int fa, int sumx, int sumy) {
        int res = w[u];
        for (int i = h[u]; ~i; i = ne[i]) {
            int j = e[i];
            if (j == fa) continue;
            int t = dfs(j, u, sumx, sumy);
            res ^= t;
            // 如果是第二次断开线的dfs，则求最小值
            if (sumx != -1) {
                int a[3] = {t, sumx ^ t, sumy};
                sort(a, a + 3);
                ans = min(ans, a[2] - a[0]);
            }
        }
        return res;
    }
    int minimumScore(vector<int>& nums, vector<vector<int>>& edges) {
        w = nums;
        int n = nums.size();
        // 枚举第一次断开的线
        for (int i = 0; i < edges.size(); i ++) {
            memset(h, -1, 4 * n);
            idx = 0;
            // 建树
            for (int j = 0; j < edges.size(); j ++) {
                if (j != i) {
                    int a = edges[j][0], b = edges[j][1];
                    add(a, b), add(b, a);
                }
            }
            int x = edges[i][0], y = edges[i][1];
            int sumx = dfs(x, -1, -1, -1);
            int sumy = dfs(y, -1, -1, -1);
            // 依次遍历两棵树中其余线
            dfs(x, -1, sumx, sumy);
            dfs(y, -1, sumy, sumx);
        }
        return ans;
    }
};
```

## 300场周赛

### [6109. 知道秘密的人数](https://leetcode.cn/problems/number-of-people-aware-of-a-secret/)

**题意：**

在第 1 天，有一个人发现了一个秘密。

给你一个整数 delay ，表示每个人会在发现秘密后的 delay 天之后，每天 给一个新的人 分享 秘密。同时给你一个整数 forget ，表示每个人在发现秘密 forget 天之后会 忘记 这个秘密。一个人 不能 在忘记秘密那一天及之后的日子里分享秘密。

给你一个整数 n ，请你返回在第 n 天结束时，知道秘密的人数。由于答案可能会很大，请你将结果对 10^9 + 7 取余 后返回。

**思路：**

周赛时候实现的方法，用f存储当前知道秘密的人数，add存储该天增加的人数，sadd存储增加人数的增速。

关于这个增速，当时想法就是差分的一种思路，当前天增加的人数，在delay天之后增速会加上人数的个数，在forget天之后，增速会减去人数的个数，利用add加上sadd获得当天实际增加的人数。当然，在forget天之后，还要减去当前天增加的人数。

但是周赛的时候，能通过大部分样例，有个别样例出现了负值，后面才想明白，在对值取mod的时候有的结果会是负值，这样最终结果也会变为负值，先加上mod再取模就可以解决这个问题。

```
const int N = 2010, mod = 1e9 + 7;
typedef long long LL;
LL f[N], sadd[N], add[N];
class Solution {
public:
    int peopleAwareOfSecret(int n, int delay, int forget) {
        memset(add, 0, sizeof add);
        memset(sadd, 0, sizeof sadd);
        memset(f, 0, sizeof sadd);
        f[1] = 1, sadd[delay + 1] += 1, sadd[forget + 1] -= 1, f[forget + 1] -= 1;
        for (int i = 2; i <= n; i ++) {
            add[i] = (sadd[i] + add[i - 1]) % mod;
            f[i] = (f[i] + f[i - 1] + add[i]) % mod;
            if (add[i])  {
                sadd[i + delay] = (sadd[i + delay] + add[i]) % mod;
                sadd[i + forget] = (sadd[i + forget] - add[i]) % mod; 
                f[i + forget] = (f[i + forget] - add[i]) % mod;
            }
        }
        return (f[n] + mod) % mod;
    }
};
```

