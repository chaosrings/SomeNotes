---
title: SamplingInCG
date: 
categories:
tags:
---

## Inverse Transform Sampling

一个很重要的数学知识是逆变换采样，在算各种数值积分时会经常用到。众所周知计算机在生成随机数时只能生成均匀分布的随机数，但在进行重要性采样时需要得到密度函数$pdf(x)=f(x)$的采样分布，这里就需要用到逆变换采样。
具体的做法是

1.求累计密度函数$F_X(x)$，也就是$\int_{-\infty}^{x}f(x)dx$

2.计算$F_X(x)$的反函数$F_X^{-1}(x)$

3.生成区间$[0,1]$内的均匀分布采样点$U_i$

4.计算$X_i=F_X^{-1}(U_i)$，$X_i$也就是密度函数为$f(x)$的分布

证明过程为：

设有分布$U$~$unif[0,1]$以及变换函数$T(U)=X$，经过变换后$X$满足累计密度函数$F_X(x)$，根据定义

$F_X(x)=P(X\leqslant x)=P(T(U)\leqslant x)=P(U\leqslant T^{-1}(x))$

而$U$是$[0,1]$内的均匀分布，$P(U\leqslant u)=u$，也就是$F_X(x)=P(U\leqslant T^{-1}(x))=T^{-1}(x)$

$F_X(x)=T^{-1}(x)$也就是$T(x)=F_X^{-1}(x)$