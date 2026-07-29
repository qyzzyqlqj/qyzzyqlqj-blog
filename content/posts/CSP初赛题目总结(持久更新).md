---
author: [""]
title: "CSP初赛题目总结(持久更新)"
date: "2026-07-29"
description: ""
summary: ""
tags: ["初赛", "算法", "总结反思", "编程"]
categories: ["编程"]
series: ["编程"]
ShowToc: true
TocOpen: false
draft: false
---


## 一、常识问题

对于如下代码段，其时间复杂度为（ ）。 
 
```cpp
int s = 0;
for (int i = 1; i <= n; i *= 2)
    for (int j = 1; j <= i; j++)
        s++;
```
A. $O(n)$  
B. $O(n \log n)$  
C. $O(n^2)$  
D. $O(\log n)$ 

#### 【错选】：B
#### 【正解】：A
### 【题目考点】：时间复杂度分析
### 【错因】：没有正确理解算法时间复杂度的分析，没有带入验证盲目猜测，经验注意
### 【分析】：
1. 外层循环变量 $i$ 的取值为 $1, 2, 4, 8, \dots, 2^k$（且 $2^k \le n$）。
2. 内层循环运行 $i$ 次，因此总的执行次数为 $1 + 2 + 4 + \dots + 2^k$。这是一个等比数列求和，结果为 $2^{k+1} - 1 \approx 2n$。忽略常数后，时间复杂度为 $O(n)$。

---





## 二、算法



## 三、













