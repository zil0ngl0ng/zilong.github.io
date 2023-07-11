---
title: 筛选n以内的质数的一些方法
date: 2023-7-3 15:46:06
categories:
- 算法
tags:
- 质数
mathjax: true
---
筛出从1到 $n$ 之间的素数的算法，一些常见的方法以及理解。
<!-- more -->


## （1）普通筛选

最普通的筛法，也就是将前 n 个正整数一个一个来判断是否为素数，并且在判断素数的时候要从 2 枚举到 这个数−1来判断。时间复杂度为$O(n^2)$。

```c++
for(int i=1;i<=n;++i)//枚举1到n
{
    bool flg=0;
	for(int j=2;j<i;++j)//枚举2到i
	{
		if(i%j==0)//如果i%j=0,也就是i已经不为素数了
		{
			flg=1;//打上标记
			break;//跳出循环，不用再枚举了
		}
	}
	if(flg==0)//如果没有被打上标记
	prime[i]=1;//prime来标记这个数是否为素数。
}
```

## （2）普通筛选的优化

在判断素数的时候，不一定需要枚举到$i-1$，只需要枚举到$\sqrt{i}$ 。时间复杂度为$O(n\sqrt{n})$。

```c++
for(int i=1;i<=n;++i)//枚举1到n
{
    bool flg=0;
	for(int j=2;j*j<=i;++j)//枚举2到i
	{
		if(i%j==0)//如果i%j=0,也就是i已经不为素数了
		{
			flg=1;//打上标记
			break;//跳出循环，不用再枚举了
		}
	}
	if(flg==0)//如果没有被打上标记
	prime[i]=1;//prime来标记这个数是否为素数。
}
```

## （3）埃氏筛

我们发现，上面两种筛法会筛到许多没有意义的数，所以我们必须换一种思想方式。

埃氏筛，就是先将 $prime$ 数组全部赋值为 $1$ (记得将 $prime[1]$ 赋值为 $0$ )。
仍然是要从 $1$ 枚举到 $n$  。我们先假设当前枚举到了 $i$ ，如果 $prime[i]=1$ ，也就是 $i$ 为质数，则我们可以知道 $i$ 的倍数均为合数，所以我们就将 $prime[i*k]$ 赋值为 $0$ 。最终筛完之后，如果 $prime[i]=1$ ，$i$ 就是质数。

时间复杂度为 $O(nloglogn)$。

```c++
memset(prime,1,sizeof(prime));
priem[1]=0;
for(int i=1;i<=n;++i)
{
	if(prime[i])
	{
		for(int j=2;j*i<=n;++j)
		prime[i*j]=0;
	}
}
```

## （4）欧拉筛（线性筛）

我们发现，埃氏筛已经很快了，但是还是有所不足。

因为在埃氏筛中，有很多数有可能被筛到很多次（例如 6, 他就被 2 和 3 分别筛了一次）。 所以在欧拉筛中，我们就是在这个问题上面做了优化，使得所有合数只被筛了一次。

首先，我们定义 $isNotprimes[i]$ 数组表示 $i$ 是否为合数，$primes$ 储存已经找到的所有质数。

如果当前已经枚举到了 $i$ ，如果 $isNotprimes[i]=0$ ，也就是 $i$ 为质数。则 $primes[cnt+1]=i$。

然后我们每一次枚举都要做这个循环：枚举 $j$ ，$st[primes[j]*i]=1$ 。（因为 $primes[j]$ 为质数， $i$ 就表示这个质数的多少倍，要把他筛掉，$primes[j]$是元要筛选掉的合数的最小质因子）。

**注意，接下来是重点！** 如果 $k$  $mod$  $primes[j] = 0$ ，跳出第二层循环。(因为欧拉筛默认每一个合数只能由他的最小质因数筛去，而满足以上条件之后，$primes[j]$ 就不是这个数字的最小质因数了，所以我们跳出第二层循环)。 因此，有了这一层优化之后，每一个合数就只能被筛掉一次了。

复杂度为 $O(n)$ 。

```c++
class Solution {
public:
    int countPrimes(int n) {
        vector<bool> isNotprimes(n);                    // 标记是否为合数
        vector<int> primes;                             // 存遍历到的质数，使用vector会很慢
        int ans = 0;
        for (int i = 2; i < n; ++i) {
            if (!isNotprimes[i]) {                      // 质数
                ++ans;
                primes.push_back(i);                    // 添加到质数数组
            }
            /**
             * 标记i的倍数为合数（筛掉）：“j < primes.size()”条件可以省略，因为：
             *  - 如果i是质数，那么if语句的条件一定在primes最后一个元素成立，因为primes里最后一个数为刚加进去的i
             *  - 如果i是合数，那么一定会小于它的（已在primes中的）质数整除，primes里有他的质因子，所以if会成立
             */
            for (int j = 0; (long long)i * primes[j] < n; ++j) {
                isNotprimes[i * primes[j]] = true;  // 如果i为合数，且 “i的最小质因子*i” 未越界，则 “i的最小质因子*i” 得到的合数一定能被筛掉
                /**
                 * - 此if条件判断的作用：primes数组中的质数是递增的，当 “i%primes[j]==0” 成立（i能被primes[j]整除），
                 * 那么 “i*primes[j+1]” 这个合数肯定也可以被 “primes[j]” 筛掉，
                 * 因为i中的含有 “primes[j]” ，也就是 “i*primes[j+1]=(k*primes[j])*primes[j+1]”（其中k为i的另一个因子），
                 * 所以，当 “i%primes[j]==0” 成立时，需要终止，否则当i遍历到 “k*primes[j+1]” 时，
                 * “i*primes[j+1]” 也就是 “(k*primes[j])*primes[j+1]” 会再次被筛掉。
                 */
                /**
                 * - 那当前遍历的i如果是合数，会不会在之前没被筛掉？
                 * 答：不会，假设一个合数为z，它的最小质因子为x，z=x*y（其中，y为另一个因子，它可能为质数也可能为合数），必有x<=y，
                 * 由于x>1，因此有 y<z，当i遍历到y，分两种情况：
                 * （1）当y质数，那么 “i%primes[j]==0” 的成立条件为 “primes[j]=y”，因此x必被primes[j]遍历到，也就是z必被筛掉。
                 * （2）当y为合数，那么此时y可类比成当前的z，也可分解成两个数 y=x_y*y_y（下划线后面的y表示是y的因子，x_y为y的最小质因子），
                 *      由于任何合数都能被分解成多个质因子相乘，y即是合数也是z的因子，因此y的因子也是z的因子，
                 *      由于x是z的最小质因子，因此必有x<=x_y。
                 *      由于对于一个合数y，只要 “y*x_y<n” ，也就是循环的条件成立（不越界），那么 x_y 必被primes[j]遍历到，
                 *      而 x<x_y，primes数组又是从小到大遍历，因此 x 必被primes[j]遍历到，也就是z必被筛掉。
                 */
                if (i % primes[j] == 0) {               // 整除中断，保证合数被最小的质因数筛掉
                    break;
                }
            }
        }
        return ans;
    }
};
```
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="./Primes/线性筛.png" width = "60%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 1: 线性筛过程图解
  	</div>
</center> 

（1）线性筛图解举例说明为什么中断筛选

图解模拟如上，当 $i=4$ 时，遍历 $primes$ 中的质数，当遍历到 $2$ 时，$8=4 \times 2$ 被筛选掉了，而且因为 $4\%2==0$ 成立，所以停止筛选，令 $i=5$ 继续筛选。 

如果，$i=4$ 时，遍历 $primes$ 到 $2$ 以后继续遍历，即遍历到 $3$ ，此时要筛选掉的数字为 $12=4\times 3$ ，但是 $4=2\times 2$，所以 $12=4\times 3=2\times 2\times 3=6\times 2$，所以当 $i=6$ 时，遍历 $primes$ 到 $2$ 的时候自然会筛选掉，会重复筛选，所以 $i=4$ 时，遍历 $primes$ 到 $2$ 以后停止遍历。

（2）那当前遍历的 $i$ 如果是合数，会不会在之前没被筛掉？

代码开头，$isNotprimes[i]$ 为 $false$ ，直接认定 $i$ 为质数，是否合理？是否在遍历到 $i$ 之前将其筛选掉？

假如 $i$ 为合数，其最小因子为 $x$，另一个因子为 $y$ ，那么$i=x\times y$。因为最小因子为 $x$，所以 $y\ge x$ ，所以 $i>y \ge x$，所以当 $i$ 遍历到 $y$ 的时候，遍历 $primes$ 到 $x$ 的时候，肯定会把 $x\times y$ 筛选掉；