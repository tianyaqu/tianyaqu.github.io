---
layout: post
title: "Monte Carlo模拟"
date: 2015-05-17 14:30
comments: true
categories: [金融]
tags: [Monte Carlo,资产定价]
---
Monte Carlo在金融上的应用，一直想找个例子，百度给出的结果却都是老生常谈的算pi的例子，甚至李洋那本matlab量化也是如此。顺便批评一下这本书立意太低，matlab用法占了1/4,期权定价再重复一下赫尔书中的知识，最终竟然以均线交叉策略作例子结束。所以下面给出搜集的几个Monte Carlo在金融中的应用例子，以飨诸位。

文中的思路来源于Columbia 大学 Martin Haugh 的[蒙塔卡罗模拟讲义](http://www.columbia.edu/~mh2078/MonteCarlo.html)，如有理解不到位的地方，请指正。

<!--more-->

蒙塔卡罗是可以看作是大数定律的具体应用。前提条件是，要生成的千千万万的随机值，对于这些随机值的用法，有两种选择。一种是投点法，一种是期望法。投点法就是pi的例子，在2x2的方格里生成2n个随机值(n个随机点P(xi,yi)，也即是个2n个)，计算落入圆内外的点数目之比即为4/pi的估计。
<p>期望法与此稍有不同，它只要n个随机值即可，计算f(x)的算是平均即是所求。原理是这样的：随机变量 X 服从[0,1]上的均匀分布，则Y=f(X)的数学期望为  <center>$E(f(X)) = \int^1_0 f(x)dx = J$  </center>所以估计J的值就是估计f(X)的数学期望值，由辛钦大数定律，可以用f(X)的观察值的均值取估计f(X)的数学期望。下面用几个例子来阐述</p>

#### 1.积分计算 
<p> 算阴影面积,其中$f = \int^1_0 x^3 dx$ ，用上述的期望法计算</p>
```
n=500;
x = rand(n,1);
g = x.^3;
estimate = mean(g);
% or simply
% estimate = mean(rand(100,1).^3);
```

### 2.资产定价
<p>假定某股票价格遵从几何布朗运动，那么股票在任意时刻T的价格可用 SDE(stochastic differential equation)解出，它的解析解为
<center>$S_T = S_0 e^{(µ−\sigma^2/2)T+\alpha B_T}$</center>
u,\sigma为收益率、波动率，$B_T$代表着维纳过程，即高斯随机量(期望为0的正态分布)。与该股票关联的一衍生品，如果我们知道(假定)在时刻T收益为H(X),根据风险中性原则，它的初始价格
$h_0$满足
<center>$h_0 = E_{0}^{Q}[e^{-rT}h(X)]$</center>
也即衍生品的价格(解析解)。</p>
<p>
OK，现在我们要计算一执行价K，到期时间为T的欧式期权的价格$C_0$。首先，利用上文的结论，可以得到$C_0$为：
<center>$C_0 = e^{-rT}E_{0}^{Q}[max(0,S_{0}e^{(r-\sigma^2/2)T+\sigma B_{T}}-K)]$</center>
E_{0}^{Q}[.]表示其期望。
</p>
那么，期权价格的数值解计算过程为

```
set sum = 0
for i = 1 to n
    %代入解析表达式求值
    generate S_T
    set sum = sum + max (0, S_T − K)
set Cc0 = e_{−rT}( sum/n)
```
### 3.组合收益率
根据组合内股票相关与不相关，可以分成两个问题。先从简单的不相关说起。
#### 3.1 独立组合
<p>现在我们有多只股票，这里以两只为例，分别为A,B，两者相互独立，都服从几何布朗运动。$S_{a}(t)$,$S_{b}(t)$为A,B在t时刻价格，开始时候(t = 0)，我们购买了$n_a$单位A,$n_b$单位B，此时组合价值为
<center>$W_T = n_{a}S_{a}(T) + n_{b}S_{b}(T)$</center>
那么，我们把这些代入Monte Carlo框架中来
</p>
```
sum = 0
for i = 1 to n
    generate $X_i = (S_{a}^{i}(T),S_{b}^{i}(T))$
    $sum += W_T$
$E(W_T) = sum/n$
r = E(W_T)/W_0 - 1
```
#### 3.2 相关组合
<p>下面讨论相关的情况。首先要明确怎么才算相关？对于几何布朗运动来讲，相关用$B_t$项的协方差来度量。这个量是个服从正态分布的变量(不要认为随机就独立了，只有为0时候才认为它们独立)，用协方差来度量他们的相关程度。由于该项是股价的动力，股价也作用于收益率上，所以用收益率来衡量两只股票的相关性。
</p>
<p>
其次，要利用一个结论。概率论上讲过，多个正态分布的线性组合仍然是正态分布，那么多只相关股票的联合效果为($B_t$项是期望为0的正态分布)：
<center>$c_1Z_1 + ... c_nZ_n \sim N(0,\sigma^2)$</center>
这个式子要反过来看，即在已知组合的协方差矩阵后(收集组合内各只股票的历史收益率数据，计算协方差得到)，如何获得各个分布的系数(或权重)$c_n$？
</p>
<p>可喜的是，数学系的人帮我们在这里接下了担子。首先将问题的描述更具体化:
<center>$C^TZ \sim N(0,C^TC)$</center>
也就是，只要能获得$C$,使得$C^TC = \sum$,那么就认为找到了前面要用的系数$c_n$,${\sum}$为协方差矩阵,而这一步，只要用Cholesky分解即可得出。
</p>

```
function[theta] = portfolio_evaluation(mua,mub,siga,sigb,n,T,rho,S0a,S0b,na,nb);
% This function estimates the probability that the portfolio value, W_T, falls
% by more than 10%. n is the number of simulated values of W_T that we use.
W0 = na*S0a + nb*S0b;
Sigma = [siga^2 siga*sigb*rho;
siga*sigb*rho sigb^2];
B = randn(2,n);
C = chol(Sigma);
V = C’ * B;
STa = S0a * exp((mua - (siga^2)/2)*T + sqrt(T)*V(1,:));
STb = S0b * exp((mub - (sigb^2)/2)*T + sqrt(T)*V(2,:));
WT = na*STa + nb*STb;
theta = mean(WT/W0) - 1;
```
