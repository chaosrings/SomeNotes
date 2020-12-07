---
title: SamplingInCG
date: 
categories:
- cgmath
mathjax: true
---

## Inverse Transform Sampling

一个很重要的数学知识是逆变换采样，在算各种数值积分时会经常用到。众所周知计算机在生成随机数时只能生成均匀分布的随机数，但在进行重要性采样时需要得到密度函数$pdf(x)=f(x)$的采样分布，这里就需要用到逆变换采样。
具体的做法是

1.求累计密度函数$F_X(x)$，也就是$\int_{-\infty}^{x}f(x)dx$

2.计算$F_X(x)$的反函数$F_X^{-1}(x)$

3.生成区间$[0,1]$内的均匀分布采样点$U_i$


4.计算$X_i=F_X^{-1}(U_i)$，$X_i$也就是密度函数为$f(x)$的分布

ps:计算反函数$y=f(x)$，替换$xy$位置$x=f(y)$再次推导$y$的解析式就是反函数$f^{-1}(x)$了。

证明过程为：

设有分布$U$~$unif[0,1]$以及变换函数$T(U)=X$，经过变换后$X$满足累计密度函数$F_X(x)$，根据定义

$F_X(x)=P(X\leqslant x)=P(T(U)\leqslant x)=P(U\leqslant T^{-1}(x))$

而$U$是$[0,1]$内的均匀分布，$P(U\leqslant u)=u$，也就是$F_X(x)=P(U\leqslant T^{-1}(x))=T^{-1}(x)$

$F_X(x)=T^{-1}(x)$也就是$T(x)=F_X^{-1}(x)$

## Importance Sampling

重要性采样需要生成与密度函数分布相同的采样点，那么就需要逆变换采样了。

以法线半球面均匀分布为例，其密度函数$pdf(h)=\frac{n·h}{\pi}$

用球面坐标可以表示为$pdf(\theta,\phi)=\frac{cos\theta sin\theta}{\pi}$

(这里可以理解为法线$n$是球坐标系的z轴，因此有$cos\theta=n·h$)

$pdf_{\Theta}(\theta)=\int_0^{2\pi}pdf(\theta,\phi)d\phi=2cos\theta sin\theta$

$cdf_{\Theta}(\theta)=sin^2\theta$

$pdf_{\Phi|\Theta}(\phi|\theta)=\frac{pdf(\theta,\phi)}{pdf(\phi)}=\frac{1}{2\pi}$

$cdf_{\Phi|\Theta}(\phi|\theta)=\frac{\phi}{2\pi}$

计算得到$cdf_{\Theta}(\theta)$以及$cdf_{\Phi|\Theta}$，就可以使用逆变换采样了，用[Hammersley](https://blog.csdn.net/i_dovelemon/article/details/76599923)采样生成二维$[0,1]$均匀分布$X=(X_0,X_1)$

$cdf_{\Phi|\Theta}^{-1}(\phi)=2\pi \phi$

$\Phi_i=cdf_{\Phi|\Theta}^{-1}(X_0)=2\pi X_0$

$cdf_{\Phi|\Theta}^{-1}(\theta)=arcsin\sqrt{\theta}$

$\Theta_i=cdf_\Theta^{-1}(X_1)=arcsin\sqrt{X_1}$

最终得到的$(\Theta_i,\Phi_i)$就是法线半球面均匀分布的采样点，可以用这些采样点来计算表面的Irradiance...

PBR着色模型中的Specular BRDF类似，中间向量的分布密度函数$pdf(h)=D(h)(n·h)$，用球面坐标可以表示为$pdf(\theta,\phi)=D(h)cos\theta sin\theta=\frac{\alpha^2cos\theta sin\theta}{\pi(cos^2\theta(\alpha^2-1)+1)^2}$


$pdf_{\Theta}(\theta)=\int_0^{2\pi}pdf(\theta,\phi)d\phi=\frac{2\alpha^2cos\theta sin\theta}{(cos^2\theta(\alpha^2-1)+1)^2}$

$cdf_{\Theta}=\int_0^\theta pdf(\theta)d\theta=\frac{\alpha^2sin2\theta}{\pi((\alpha^2-1)cos^2\theta+1)^2}$

$pdf_{\Phi|\Theta}(\phi|\theta)=\frac{pdf(\theta,\phi)}{pdf(\phi)}=\frac{1}{2\pi}$

$cdf_{\Phi|\Theta}(\phi|\theta)=\frac{\phi}{2\pi}$



$\Phi_i=cdf_{\Phi|\Theta}^{-1}(X_0)=2\pi X_0$

$\Theta_i=cdf_\Theta^{-1}(X_1)=cos^{-1}\sqrt\frac{1-X_1}{(\alpha^4-1)X_1+1}$

