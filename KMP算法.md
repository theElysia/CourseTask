## KMP算法讲解
#### author: Zevick

前言：
写一下自己的理解思路，分2步完成整个KMP算法。

KMP算法思路是顺序逐字符匹配target字符串与text文本，但当某一字符匹配失败时，利用target本身的结构，可以相比最naive的匹配跳过一些不必要的比较。

### Step 1: 计算最长前后缀字符串
这个步骤就是分析target字符串本身的结构。当target存在类似aabaaa的时候，若最后一个a与text不匹配，我们可以复用之前已匹配的aabaa的信息，这里就需要用到前后缀字符串。数学的定义为：

$$
prefix[i]=
\begin{cases}
0 \quad i=0\\
\max _ {j\lt i} j \quad st. pattern[0:j-1]=pattern[i-j+1:i] \\
\end{cases}
$$

##### 注意这个最长前后缀字符串不能为字符串本身，故有约束$prefix[i]\leq prefix[i-1]+1\leq i$。
注意在求解 $prefix[i]$时可以利用 $prefix[0:i-1]$的信息，比较容易得到prefix的递推式。下面给出求prefix的代码：

```c++
int *buildNext(const char *pattern)
{
    size_t size_pattern = strlen(pattern);
    int *prefix = new int[size_pattern];
    int t = prefix[0] = 0;
    for (size_t i = 1; i < size_pattern; i++)
    {
        // 注意每次迭代t的初值为 prefix[i-1]
        while (t > 0 && pattern[t] != pattern[i])
        // 递归过程类似 (aabaabaa)->(aabaa)->(aa)
            t = prefix[t - 1];
        if (pattern[t] == pattern[i])
            prefix[i] = ++t;
        else
            prefix[i] = 0;
    }
...
```


### Step 2: 构建next列表

显然直接根据 $prefix[i]$来匹配text仍不是最优，比如对于串aabaab，$prefix[5]=3$，但当最后一个b不匹配时，我们并不需要去接着 $pattern[0:2]$继续匹配。

在 $target[i]\neq text[j]$ 字符不匹配时，我们希望的next列表有如下性质：

- $next=-1$ ，当下一次匹配该尝试 $target[0], text[j+1]$ 时
- $next=t\neq -1$ ，当下一次匹配该尝试 $target[t], text[j]$ 时

当 $next=t$ 时，显然有 

$$
\begin{cases}
pattern[0:t-1]=pattern[i-t:i-1]=text[j-t:j-1] \\
 pattern[t]\neq pattern[i] \\
\end{cases}
$$

那么就可以根据prefix构建next，下面给出代码：

```c++
int *buildNext(const char *pattern)
{
    size_t size_pattern = strlen(pattern);
    int *prefix = new int[size_pattern];
    int t = prefix[0] = 0;
    for (size_t i = 1; i < size_pattern; i++)
    {
        while (t > 0 && pattern[t] != pattern[i])
            t = prefix[t - 1];
        if (pattern[t] == pattern[i])
            prefix[i] = ++t;
        else
            prefix[i] = 0;
    }

    int *next = new int[size_pattern];
    next[0] = -1;
    for (size_t i = 1; i < size_pattern; i++)
    {
        int t = prefix[i - 1];
        while (t > 0 && pattern[i] == pattern[t])
            t = prefix[t - 1];
        if (pattern[i] == pattern[t])
            next[i] = -1;
        else
            next[i] = t;
    }
    delete[] prefix;
    return next;
}
```

### 最后优化
把prefix与next分成2个循环分别求解有点冗余，可以合并。

首先意识到求解 $next[i]$需要参考的是 $prefix[i-1]$。
而 $next[0:i-1]$包含了 $prefix[0:i-2]$的信息，故可以只保留 $prefix[i-1]$。

此时递归求解prefix的过程用到了next的性质。注意总是有$prefix[i]\leq prefix[i-1]+1$这条约束，方便我们递推计算。相当于此时用已求解的next去匹配pattern[i-1]。

```c++
int *buildNext_ultimate(const char *pattern)
{
    size_t size_pattern = strlen(pattern);
    int *next = new int[size_pattern];
    next[0] = -1;
    int prefix = -1;
    for (size_t i = 1; i < size_pattern; i++)
    {
        // 先求prefix[i-1]
        while (prefix > -1 && pattern[prefix] != pattern[i-1])
            prefix = next[prefix];
        // 再求next[i]，注意next性质
        // 若 pattern[i] == pattern[prefix]
        // 保证了 pattern[i]！=pattern[next[prefix]]
        ++prefix;
        if (pattern[i] != pattern[prefix])
            next[i] = prefix;
        else
            next[i] = next[prefix];
        // 若这样的顺序不好理解，可以看如下代码
        // if (pattern[i] != pattern[prefix+1])
        //     next[i] = prefix+1;
        // else
        //     next[i] = next[prefix+1];
        // ++prefix;
    }
    return next;
}
```

再补上匹配的代码

```cpp
char *textMatch(char *text, const char *pattern, const int *next)
{
    char *p = text;
    size_t m = strlen(pattern);
    int t = 0;
    while (*p)
    {
        if (t == -1 || pattern[t] == *p)
        {
            ++t;
            ++p;
            if (t == m)
                return p - m;
        }
        else
            t = next[t];
    }

    return nullptr;
}
```