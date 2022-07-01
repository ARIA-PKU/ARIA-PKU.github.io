---
title: 周赛题整理
date: 2022-05-29T15:35:29+08:00
lastmod: 2022-05-29T15:35:29+08:00

cover: https://oss.surfaroundtheworld.top/blog-pictures/butterfly.jpg
# images:
#   - /img/cover.jpg
categories:
  - 周赛算法题
tags:
  - 算法题
# nolastmod: true
draft: false
---

总结整理遇到的有趣的题目。

<!--more-->

# 49周周赛

## 点的赋值

题目：https://www.acwing.com/problem/content/4418/

思路：

染色法判定二分图

很显然，任意一条路径要形成 奇数——偶数——奇数——……奇数——偶数——奇数——…… 这种情况，即将点分为奇数和偶数两部分，相当于对图进行染色，使得相邻两点颜色不同，即转换为二分图染色问题，由于可能不止一个连通块，如果存在一个连通块不能完成二分图染色则表示这样路径不存在，则方案数为 0，否则对于每一个连通块，设两种颜色的数量分别为 cnt1,cnt2，cnt1 表示为奇数数量时，每个点都有 1 或 3 两种情况，cnt2 也可以表示为奇数数量，故一个连通块内有 2<sup>cnt1</sup>+2<sup>cnt2</sup> 种方案，另外，连通块之间的方案是互不影响的，利用乘法原理即得总方案数.

时间复杂度：O(n+m)

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

# 53周周赛

## 整除子串

题目：https://www.acwing.com/problem/content/4429/

思路：**数学规律题**

存在一些小学数奥的规律（我也只知道部分，这里把别人总结的抄过来记录一下）

```
被3整除：
若一个整数的数字和能被3整除，则这个正数能被3整除
被4整除：
若一个整数的末尾两位数能被4整除，则这个数能被4整除。
被5整除：
若一个整数的末位是0或5，则这个数能被5整除。
被6整除：
若一个整数能被2和3整除，则这个数能被6整除。
被7整除：（比较麻烦一点）
若一个整数的个位数字截去，再从余下的数中，减去个位数的2倍，如果差是7的倍数，则原数能被7整除。如果差太大或心算不易看出是否7的倍数，就需要继续上述「截尾、倍大、相减、验差」的过程，直到能清楚判断为止。例如，判断133是否7的倍数的过程如下：13－3×2＝7，所以133是7的倍数；又例如判断6139是否7的倍数的过程如下：613－9×2＝595 ， 59－5×2＝49，所以6139是7的倍数，余类推。
被8整除：
若一个整数的未尾三位数能被8整除，则这个数能被8整除。
被9整除：
若一个整数的数字和能被9整除，则这个整数能被9整除。
被10整除：
若一个整数的末位是0，则这个数能被10整除。
被11整除：
若一个整数的奇位数字之和与偶位数字之和的差能被11整除，则这个数能被11整除。11的倍数检验法也可用上述检查7的「割尾法」处理！过程唯一不同的是：倍数不是2而是1！
被12整除：
若一个整数能被3和4整除，则这个数能被12整除。
被13整除：
若一个整数的个位数字截去，再从余下的数中，加上个位数的4倍，如果差是13的倍数，则原数能被13整除。如果差太大或心算不易看出是否13的倍数，就需要继续上述「截尾、倍大、相加、验差」的过程，直到能清楚判断为止。
被17整除：
若一个整数的个位数字截去，再从余下的数中，减去个位数的5倍，如果差是17的倍数，则原数能被17整除。如果差太大或心算不易看出是否17的倍数，就需要继续上述「截尾、倍大、相减、验差」的过程，直到能清楚判断为止。
若一个整数的末三位与3倍的前面的隔出数的差能被17整除，则这个数能被17整除。
被19整除：
若一个整数的末三位与7倍的前面的隔出数的差能被19整除，则这个数能被19整除。
若一个整数的个位数字截去，再从余下的数中，加上个位数的2倍，如果差是19的倍数，则原数能被19整除。如果差太大或心算不易看出是否19的倍数，就需要继续上述「截尾、倍大、相加、验差」的过程，直到能清楚判断为止。
被23整除：
若一个整数的末四位与前面5倍的隔出数的差能被23(或29)整除，则这个数能被23整除
```

根据规律，判断后两位即可。

```
#include <iostream>
using namespace std;

typedef long long LL;  // 注意不要爆int

const int N = 3e5 + 10;
char s[N];

int main() {
    scanf("%s", s);
    LL res = 0;
    for (int i = 0; s[i]; i ++) {
        if ((s[i] - '0') % 4 == 0) res ++;  // 最后一位单独判断
        if (i && (10 *(s[i - 1] - '0') + (s[i] - '0')) % 4 == 0) res += i;  // 后两位满足则以当前结尾的子串都满足
    }
    printf("%lld", res);
    return 0;
}
```

## 树中节点和

题目：https://www.acwing.com/problem/content/4430/

思路：树相关的题目

由于偶数层的节点大小没有给出，所以需要计算出使得总结点的和最小时候，偶数节点应该取的值；

对于一个偶数层节点，假设其值为p，有一个父节点，其路径sum值记为last_s，还有若干个子节点，其路径sum值分别为s1, s2, s3......

如果希望总的最小，则p就要取到可能的最大值（这里很容易理解，画图看一下就好），即$min(s1 - last_s, s2 - last_s, s3 - last_s, ......)$，并且每两层之间相互不影响，然后还需要在处理上注意一些细节。

```
#include <iostream>
#include <cstring>
using namespace std;

typedef long long LL;

const int N = 1e5 + 10, INF = 2e9;
int h[N], e[N], ne[N], idx;
int s[N];
LL ans;
bool flag;

void add(int a, int b) {
    e[idx] = b, ne[idx] = h[a], h[a] = idx ++;
}

// u为当前点编号，depth是深度，last_s是上一个节点的s值
void dfs(int u, int depth, int last_s) {
    if (depth % 2) { // 奇数层直接向下传递，只在偶数层处理
        for (int i = h[u]; ~i; i = ne[i]) {
            int j = e[i];
            dfs(j, depth + 1, s[u]);
        }
    }
    else {
        // 在偶数层，节点均为-1
        int p = INF;
        for (int i = h[u]; ~i; i = ne[i]) {
            int j = e[i];
            dfs(j, depth + 1, 0);
            p = min(p, s[j] - last_s);  // 找到当前节点的最大取值p
        }
        
        // 判断严格大于等于前一个节点的值
        if (p < 0) flag = true;
        
        for (int i = h[u]; ~i; i = ne[i]) {
            int j = e[i];
            ans += s[j] - last_s - p;
        }
        
        // 如果节点不是叶子节点则加上，叶子节点可以直接取0即可
        if (p != INF) ans += p;
    }
}

int main() {
    memset(h, -1, sizeof h);
    int n, p;
    scanf("%d", &n);
    for (int i = 2; i <= n; i ++) {
        scanf("%d", &p);
        add(p, i);
    }
    for (int i = 1; i <= n; i ++) scanf("%d", &s[i]);
    ans = s[1];
    dfs(1, 1, 0);
    if (flag) puts("-1");
    else printf("%lld\n", ans);
    return 0;
}
```

# 54周周赛

## 无线网络

https://www.acwing.com/problem/content/4432/

**思路：**

将所有点按照距离r1从远到近排序，然后依次取出当前距离r1最远的点加入r2，不断计算更新两个半径的平方和最小值。

```
#include <iostream>
#include <algorithm>
#include <cstring>

#define x first
#define y second

using namespace std;

typedef long long LL;
typedef pair<int, int> PII;

const int N = 2010;
PII q[N], s1, s2;
int n;

LL dist(PII a, PII b) {
    LL dx = a.x - b.x;
    LL dy = a.y - b.y;
    return dx * dx + dy * dy;
}

bool cmp(PII a, PII b) {
    return dist(a, s1) > dist(b, s1);
}

int main() {
    scanf("%d%d%d%d%d", &n, &s1.x, &s1.y, &s2.x, &s2.y);
    for (int i = 0; i < n; i ++) scanf("%d%d", &q[i].x, &q[i].y);
    LL res = 1e18, r = 0;
    // 按照到r1从远到近排序，每次移出最远的点加入r2
    sort(q, q + n, cmp);
    for (int i = 0; i < n; i ++) {
        res = min(res, dist(q[i], s1) + r);
        r = max(r, dist(s2, q[i]));
    }
    // 最后判断所有点在r2内的情况
    res = min(res, r);
    printf("%lld\n", res);
    return 0;
}
```

## 括号序列

https://www.acwing.com/problem/content/description/4433/

**题意：**

给定括号序列，改变一个括号使得括号匹配，求共有多少个位置的括号改成相反括号可以使得括号匹配。

**思路：**

合法的情况只有l=r+2, r=l+2两种，然后分别处理。以r=l+2为例，当右括号数量大于左括号时候，则包括当前括号在内，前面的所有右括号的位置都可以修改成左括号，然后还要验证后面的是否合法即可。

```
#include <iostream>
#include <algorithm>
using namespace std;

const int N = 1e6 + 10;
char s[N];
int n;

int cal(char s1, string s) {
    int cnt = 0, r = 0;
    for (int i = 0; i < n; i ++) {
        if (s[i] == s1) cnt ++;
        else {
            cnt --;
            r ++;
            if (cnt < 0) {
                cnt += 2;
                // 继续验证剩下的部分是否符合，不符合则返回0
                for (int j = i + 1; j < n; j ++) {
                    if (s[j] == s1) cnt ++;
                    else {
                        cnt --;
                        if (cnt < 0) return 0;
                    }
                }
                return r;
            }
        }
    }
    return 0;
}

int main() {
    scanf("%d %s", &n, s);
    int l = 0, r = 0;
    for (int i = 0; i < n; i ++) {
        if (s[i] == '(') l ++;
        else r ++;
    }
    if (r == l + 2) {
        printf("%d\n", cal('(', s));
    } else if (l == r + 2) {
        reverse(s, s + n);
        printf("%d\n", cal(')', s));
    } else {
        puts("0");
    }
    return 0;
}
```

# 55周周赛

## 方格探索

题目：https://www.acwing.com/problem/content/4484/

题意大致就是在给定带有障碍的地图上移动，在给定向左和向右走的最大步数限制的条件下能到达的位置的数量。

思路：

这道题目的思路感觉异常复杂，首先根据数据量来看，DFS以及三次及以上循环的遍历都会TLE；

起始所在的位置记为$(r, c)$，利用$dist[i][j]$来表示移动的$(i, j)$位置时候，一共向右移动的距离$s1$，通过向右移动的距离可以求得向左移动的距离$s2 = s1 - (j - c)$，找到所有s1和s2满足左右移动范围的点就是结果；

又因为权值只会是0和1（对应了右移和非右移），所以可以采用01双端队列的思路（本质上就是Dijkstra堆优化思路，因为队列中队首队尾只会相差一因此可以直接手动排序了，为什么相差一也很容易想明白）。

然后问题就解决了，分析完归纳觉又没那么复杂，想出来还是挺难的，积累一下思路吧。

```
#include <iostream>
#include <cstring>
#include <algorithm>
#include <queue>

#define x first
#define y second

using namespace std;

typedef pair<int, int> PII;

const int N = 2010;
char g[N][N];
int dist[N][N];
bool st[N][N];
int n, m, sx, sy, X, Y;

void bfs() {
    memset(dist, 0x3f, sizeof dist);
    deque<PII> q;
    q.push_back({sx, sy});
    dist[sx][sy] = 0;
    int dx[4] = {-1, 0, 1, 0}, dy[4] = {0, 1, 0, -1};
    
    while (q.size()) {
        auto u = q.front();
        q.pop_front();
        if (st[u.x][u.y]) continue;
        st[u.x][u.y] = true;
        
        for (int i = 0; i < 4; i ++) {
            int nx = u.x + dx[i], ny = u.y + dy[i];
            if (nx < 0 || nx >= n || ny < 0 || ny >= m || g[nx][ny] == '*') continue;
            int w = 0;
            // 右移权值为1
            if (i == 1) w = 1;
            if (dist[nx][ny] > dist[u.x][u.y] + w) {
                dist[nx][ny] = dist[u.x][u.y] + w;
                if (w) q.push_back({nx, ny});
                else q.push_front({nx, ny});
            }
        }
    }
}

int main() {
    cin >> n >> m >> sx >> sy >> X >> Y;
    sx --, sy --;
    for (int i = 0; i < n; i ++)
        for (int j = 0; j < m; j ++)
            cin >> g[i][j];
            
    bfs();
    
    int res = 0;
    for (int i = 0; i < n; i ++) {
        for (int j = 0; j < m; j ++) {
            int t = dist[i][j];
            if (t <= Y && t - (j - sy) <= X)
                res ++;
        }
    }
    cout << res << endl;
    return 0;
}
```

# 56周周赛

##有限小数

题目：https://www.acwing.com/problem/content/4487/

题意：

给定三个整数 p,q,b，请你计算十进制表示下的 p/q的结果在 b 进制下是否为有限小数。

思路：

当一个小数可以用b进制表示的时候，在有限次k次乘b就可以得到整数。

则先将给定的p和q约分成互质的p‘和q’，则如果满足条件，则$p' * b ^ k / q'$为整数，则只要q‘能够整除$b^k$即可。可以循环给q'和b寻找最大公约数d，然后将q’除d，直到公约数d为1或者q为1。

```
#include <iostream>
using namespace std;

typedef long long LL;

LL gcd(LL a, LL b) {
    return b ? gcd(b, a % b) : a;
}

int main() {
    int T;
    scanf("%d", &T);
    while (T --) {
        LL p, q, b;
        scanf("%lld%lld%lld", &p, &q, &b);
        LL d = gcd(p, q);
        
        // 约分q
        q /= d;
        
        // 循环约分q
        while (q > 1) {
            d = gcd(q, b);
            if (d == 1) break;
            // 这里的循环优化可以过掉最后一个测试用例
            while (q % d == 0) q /= d;
        }
        if (q == 1) puts("YES");
        else puts("NO");
    }
    return 0;
}
```

