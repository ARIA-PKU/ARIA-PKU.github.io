---
title: 周赛294 LeetCode2281
date: 2022-05-22T15:59:00+08:00
lastmod: 2022-05-22T15:59:00+08:00
# author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - 周赛
tags:
  - 单调栈
  - 前缀和
# nolastmod: true
draft: false
---

单调栈的套路常见，前缀和的找规律不熟，而且要注意边界细节

<!--more-->

#LeetCode 2281. 巫师的总力量和

 题目描述
作为国王的统治者，你有一支巫师军队听你指挥。

给你一个下标从 0 开始的整数数组 strength，其中 strength[i] 表示第 i 位巫师的力量值。对于连续的一组巫师（也就是这些巫师的力量值是 strength 的 子数组），总力量 定义为以下两个值的 乘积：

巫师中 最弱 的能力值。
组中所有巫师的个人力量值 之和。
请你返回 所有 巫师组的 总 力量之和。由于答案可能很大，请将答案对 10^9 + 7 取余 后返回。

子数组 是一个数组里 非空 连续子序列。

**样例**

```
输入：strength = [1,3,1,2]
输出：44
解释：以下是所有连续巫师组：

- [1,3,1,2] 中 [1]，总力量值为 min([1]) * sum([1]) = 1 * 1 = 1
- [1,3,1,2] 中 [3]，总力量值为 min([3]) * sum([3]) = 3 * 3 = 9
- [1,3,1,2] 中 [1]，总力量值为 min([1]) * sum([1]) = 1 * 1 = 1
- [1,3,1,2] 中 [2]，总力量值为 min([2]) * sum([2]) = 2 * 2 = 4
- [1,3,1,2] 中 [1,3]，总力量值为 min([1,3]) * sum([1,3]) = 1 * 4 = 4
- [1,3,1,2] 中 [3,1]，总力量值为 min([3,1]) * sum([3,1]) = 1 * 4 = 4
- [1,3,1,2] 中 [1,2]，总力量值为 min([1,2]) * sum([1,2]) = 1 * 3 = 3
- [1,3,1,2] 中 [1,3,1]，总力量值为 min([1,3,1]) * sum([1,3,1]) = 1 * 5 = 5
- [1,3,1,2] 中 [3,1,2]，总力量值为 min([3,1,2]) * sum([3,1,2]) = 1 * 6 = 6
- [1,3,1,2] 中 [1,3,1,2]，总力量值为 min([1,3,1,2]) * sum([1,3,1,2]) = 1 * 7 = 7
所有力量值之和为 1 + 9 + 1 + 4 + 4 + 4 + 3 + 5 + 6 + 7 = 44。
输入：strength = [5,4,6]
输出：213
解释：以下是所有连续巫师组：
- [5,4,6] 中 [5]，总力量值为 min([5]) * sum([5]) = 5 * 5 = 25
- [5,4,6] 中 [4]，总力量值为 min([4]) * sum([4]) = 4 * 4 = 16
- [5,4,6] 中 [6]，总力量值为 min([6]) * sum([6]) = 6 * 6 = 36
- [5,4,6] 中 [5,4]，总力量值为 min([5,4]) * sum([5,4]) = 4 * 9 = 36
- [5,4,6] 中 [4,6]，总力量值为 min([4,6]) * sum([4,6]) = 4 * 10 = 40
- [5,4,6] 中 [5,4,6]，总力量值为 min([5,4,6]) * sum([5,4,6]) = 4 * 15 = 60
所有力量值之和为 25 + 16 + 36 + 36 + 40 + 60 = 213。
```

**限制**
$1 <= strength.length <= 10^5$
$1 <= strength[i] <= 10^9$

**算法**
**(前缀和，单调栈) O(n)**

通过单调栈求出每个位置上，左侧第一个**小于** 当前力量值的 后一个 位置 li 以及右侧 **小于等于** 当前力量值的 前一个 位置 ri。
对于 ii，定义闭区间 [li,ri][li,ri] 为当前巫师的「管辖区间」，即包含以 ii 个巫师为最弱能力值的所有区间。
所有位置都会有一个管辖区间，每个位置的管辖区间对整体答案的贡献可以通过二级前缀和在常数时间内计算出来。
假设有了普通前缀和数组 sumsum，则第 i 个巫师的管辖区间可以拆分成如下区间和的和，[l,r],[l+1,r],…,[i,r],[l,r−1],[l+1,r−1],…,[i,r−1],…,…,[i−1,i],[i,i][l,r],[l+1,r],…,[i,r],[l,r−1],[l+1,r−1],…,[i,r−1],…,…,[i−1,i],[i,i]，将以上区间写成前缀和相减的形式，可以发现，sumr,sumr−1,…,sumisumr,sumr−1,…,sumi 出现了 i−l+1 次，−suml−1,−suml,−sumi−1−suml−1,−suml,−sumi−1 出现了 r−i+1 次。通过预处理前缀和的前缀和数组，可以快速求出以上的所有区间的和。

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

作者：wzc1995
链接：https://www.acwing.com/file_system/file/content/whole/index/content/5706079/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

