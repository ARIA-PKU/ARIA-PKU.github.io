---
title: 算法题——点的赋值
date: 2022-04-30T22:16:19+08:00
lastmod: 2022-04-30T22:16:19+08:00
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /cover/acwing_4415.jpg
# images:
#   - /img/cover.jpg
categories:
  - 算法周赛题
tags:
  - 算法题
  - 二分图
# nolastmod: true
draft: false
---

周赛算法题总结，封面来自pivix的防人画师。

<!--more-->

## 题目描述

给定一个 n 个点 m 条边的无向无权图。

点的编号为 1∼n。

图中不含重边和自环。

现在，请你给图中的每个点进行赋值，要求：

每个点的权值只能是 1 或 2 或 3。
对于图中的每一条边，其两端点的权值之和都必须是奇数。
请问，共有多少种不同的赋值方法。

由于结果可能很大，你只需要输出对 998244353 取模后的结果。

## 输入格式

第一行包含整数 TT，表示共有 TT 组测试数据。

每组数据第一行包含两个整数 n,m。

接下来 m 行，每行包含两个整数 u,v，表示点 u 和点 v 之间存在一条边。

## 输出格式

一个整数，表示不同赋值方法的数量对 998244353 取模后的结果。

## 解题思路

染色法判定二分图

很显然，任意一条路径要形成 奇数——偶数——奇数——……奇数——偶数——奇数——…… 这种情况，即将点分为奇数和偶数两部分，相当于对图进行染色，使得相邻两点颜色不同，即转换为二分图染色问题，由于可能不止一个连通块，如果存在一个连通块不能完成二分图染色则表示这样路径不存在，则方案数为 0，否则对于每一个连通块，设两种颜色的数量分别为 cnt1,cnt2，cnt1 表示为奇数数量时，每个点都有 1 或 3 两种情况，cnt2 也可以表示为奇数数量，故一个连通块内有 2<sup>cnt1</sup>+2<sup>cnt2</sup> 种方案，另外，连通块之间的方案是互不影响的，利用乘法原理即得总方案数.

时间复杂度：O(n+m)

## 代码

```
#include <iostream>
#include <cstring>
using namespace std;

typedef long long LL;

const int N = 3e5 + 10, M = 2 * N, mod = 998244353;
int col[N], h[N], e[M], ne[M], idx;
int s1, s2;

void add(int a, int b) {
    e[idx] = b, ne[idx] = h[a], h[a] = idx ++;
}

int pow2(int k) {
    int ans = 1;
    while (k --) ans = ans * 2 % mod;
    return ans;
}

bool dfs(int u, int c) {  // 染色法判断二分图
    col[u] = c;
    if (c == 1) s1 ++;
    else s2 ++;
    
    for (int i = h[u]; ~i; i = ne[i]) {
        int j = e[i];
        if (!col[j]) {
            if (!dfs(j, 3 - c)) return false;
        }
        else if (col[j] == c) return false;
    }
    return true;
}

int main() {
    int T, n, m;
    scanf("%d", &T);
    while (T --) {
        scanf("%d%d", &n, &m);
        memset(h, -1, 4 * (n + 1));  // 只初始化前n个点即可
        memset(col, 0, 4 * (n + 1)); // 只初始化前n个点即可
        idx = 0;
        while (m --) {
            int a, b;
            scanf("%d%d", &a, &b);
            add(a, b), add(b, a);
        }
        
        int res = 1;
        for (int i = 1; i <= n; i ++) {
            if (!col[i]) {
                s1 = s2 = 0;
                if (dfs(i, 1)) {  // 将每个连通块方案数累乘
                    res = (LL)res * (pow2(s1) + pow2(s2)) % mod;
                }
                else {  // 不是二分图则方案数为0
                    res = 0;
                    break;
                }
            }
        }
        printf("%d\n", res);
    }
    return 0;
}
```



