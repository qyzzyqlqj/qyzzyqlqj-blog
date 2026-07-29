---
author: ["qyzzyqlqj"]
title: "浅谈C++中的超快哈希表（__gnu_pbds::gp_hash_table）"
date: "2026-07-29"
description: ""
summary: ""
tags: ["编程", "算法", "哈希", "拓展", "数据结构", "教程", "使用方法"]
categories: ["编程"]
series: ["编程"]
ShowToc: true
TocOpen: false
draft: false
---

今天(2026.7.29)上课时，听说了一个非常快的扩展库的hash表，***其空间复杂度略大，但是比`unordered_map`能快10多倍***，他就是很小众的一个扩展库`pb_ds`(`Policy-Based Data Structures`)，封装在`Mingw-G++`的扩展库中，虽然并不是标准`C++14`中的内容，但是**竞赛是可以用的**（doge），浅浅记录一下其用法。

---

`pd_ds`库中其实封装了四种容器：平衡树，字典树，哈希表，和堆。常数都很小，但是据神犇所说，只有哈希表比较有用，且只有哈希表中的`gp_hash_table`块，另一个`cc_hash_table`有时甚至不如普通的`map`，所以只介绍`gp_hash_table`。

---

## 食用方法

首先，要使用`gp_hash_table`，要先引用头文件，其**并不包含在标准库的万能头中**，不过不用担心，扩展库也有其万能头文件头文件：

`bits/extc++.h`

可以像这样引入：

~~~cpp
#include<bits/extc++.h>
~~~

接下来就可以愉快的使用了

---

### Attention：

使用`gp_hash_table`有一个注意事项，那就是的扩展库库函数有的函数如优先队列其实是跟标准库重名了，所以包含在不同的命名空间中，在这里**强烈建议手动打命名空间而不是直接`using`整个命名空间**

---

使用以下语句创建一个哈希表：

```cpp
__gnu_pbds::gp_hash_table <T1, T2> H1;
```

其中T1，T2是类型名，**注意：这里的Key也就是T1不能是pair类型**



初始化完了之后，其使用方法与`unordered_map`略有不同：

**主要常用操作区别：`gp_hash_table`并没有`.count(Key)`函数**

所以查询是否存在就只能使用如下写法：

~~~cpp
mp.find(x) != mp.end() //元素存在
~~~

其他的都大差不差

---

### Extension



#### 1.关于pair类型

如果需要使用pair类型作为键，可以用`Template`显式重载`hash`:

通用重载如下：

~~~cpp
#include <bits/stdc++.h>
#include <bits/extc++.h>
using namespace std;
template <class T1, class T2> 
struct tr1::hash <pair <T1, T2> > {
    size_t operator() (pair <T1, T2> x) const {
        tr1::hash <T1> H1; tr1::hash <T2> H2;
        return CustomHashFunc // 你自定义的 hash 函数。 如：H1(x.first) ^ H2(x.second);
    }
};
~~~

重载后，像这样的代码就可以通过编译了：

~~~cpp
//using namespace __gnu_pbds; 建议不要这样写，有可能定义比如 priority_queue 时会报错
//gp_hash_table <pair <string, pair <int, int> >, int> Table;
__gnu_pbds::gp_hash_table <pair <string, pair <int, int> >, int> Table;
~~~

---



#### 2.老生常谈的卡常问题



事实上，虽然`gp_hash_table`很快且很严格，但是在由于**其固定且开源的 hash 方式，在面对更严格的数据时仍可能会被卡**

**原理：**`gp_hash_table`的哈希方式是模 $2^{k}$ ，所以**直接造一堆模  $2^{k}$ 同余的数即可卡掉，退化成$O(N^2)$**

**拓展：**`unordered_map`为什么会被卡？

*   `unordered_map` 使用的是将键值模一个质数的哈希方式，其中质数来自一个质数表 `__prime_list`，发现其中只有 [256 个质数](https://www.luogu.com.cn/paste/3vuhvyv4)。
*   但是据说在不同的地方选的质数不一样，那不管你选啥质数了，直接造四组数据，每组数据选其中的 64 个，插入其倍数即可卡掉



通过收集网络，发现三种防止卡的方式：

##### 1.

~~~cpp
#include <ctime>
#include <ext/pb_ds/assoc_container.hpp>
const int RANDOM = time(NULL);		//更建议使用梅森旋转随机
struct MyHash {int operator() (int x) const {return x ^ RANDOM;}};
__gnu_pbds::gp_hash_table <int, int, MyHash> Table;
~~~

很显然，优点是很简单，但是缺点是任然有缺陷：

由于是模  $2^{k}$，相当于取二进制后 k 位，所以**异或一个随机数并不改变 hack 数据模  $2^{k}$ 同余的性质。**

所以请出第二种：



##### 2.

CodeForce上有篇文章作者使用了 `splitmix64` 作为哈希函数：

~~~cpp
struct custom_hash {
    static uint64_t splitmix64(uint64_t x) {
        // http://xorshift.di.unimi.it/splitmix64.c
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }

    size_t operator()(uint64_t x) const {
        static const uint64_t FIXED_RANDOM = chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + FIXED_RANDOM);
    }
};
~~~

但是这里的三个神秘数字有点难背，有什么更好记的写法吗？

有的兄弟，有的。我们直接用随机数替代神秘数字：

~~~cpp
mt19937_64 rnd(chrono::steady_clock::now().time_since_epoch().count());
struct Hsh{
	ull operator ()(ull x)const{
		static const ull s1=rnd(),s2=rnd(),s3=rnd();
		x+=s1;
		x=(x^(x>>33))*s2;
		x=(x^(x>>30))*s3;
		return x;
	}
};
gp_hash_table<ull,ull,Hsh>f;
~~~

这里使用了移位操作，但据Gemini所说，仍然有万分之一的可能会被卡掉（真卡掉给你了兄弟（bushi））

主要还有原因是不太好背，所以给出第三种方法，既好背又安全：

##### 3.

~~~cpp
struct custom_hash {
    size_t operator()(uint64_t x) const {
        static const uint64_t R = chrono::steady_clock::now().time_since_epoch().count();
        x ^= R;
        x ^= x >> 30;  					// 加上混淆：高位比特和低位比特充分混合
        x *= 0xbf58476d1ce4e5b9ULL; 	// 任意大奇数均可，破坏低位同余
        return x;
    }
};
~~~

（感谢Gemini提供的代码）

---

#### Example

使用 `gp_hash_table` 完成 [【P2580 于是他错误的点名开始了 】](https://www.luogu.com.cn/problem/P2580)（185 ms, [record](https://www.luogu.com.cn/record/289460511)）Code：

~~~cpp
#include <bits/stdc++.h>
#include <bits/extc++.h>
using namespace std;

const int MENTIONED = 1, REPEATED = 2;
string s;
int n, m;
__gnu_pbds::gp_hash_table <std::string, short> mp;


int main() {
    ios::sync_with_stdio(0);
    cin.tie(0), cout.tie(0);
    
    cin >> n;
    while (n --) 
        cin >> s, mp[s] = MENTIONED;
    
    cin >> m;
    while (m --) {
    	cin >> s;
    	if (mp[s] == MENTIONED)
            cout << "OK\n", mp[s] = REPEATED;
    	else if (mp[s] == REPEATED) 
            cout << "REPEAT\n";
    	else 
            cout << "WRONG\n";
    }
	return 0;
}
~~~

好了少年，开背吧TAT

---

## 结语

**愿世上没有卡常！！！**







