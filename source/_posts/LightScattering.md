---
title: LightScattering
date: 
categories:
- Volumetric Rendering
mathjax: true
---

## Participating Media Material

光的散射主要出现在参与介质的渲染中，关注点在于光子与组成参与介质的粒子进行交互的结果。光子在传播过程中遇到参与介质时，会有四种情况：

![EventsInMedia](/source/_posts/LightScattering/EventsInMedia.png)

a.Absorption 

光子被吸收转化为其他形式的能量，用$\sigma_a$来表示。

b.Out-scattering 

光子被反射到当前光线传播路径之外的其他方向，类似不透明表面用BRDF来描述反射光线方向的分布，参与介质则用相函数(phase function)来描述反射光线方向的分布，用$\sigma_s$来表示。

c.Emission 

当介质达到比较高的温度时，会发射出由热能转化来的光子。

d.In-scattering 

既然Out-scattering会将当前传播路径的光子反射到其他方向，同理其他传播路径的光子也能反射到当前传播路径，并对最终的Radiance产生贡献，用$\sigma_s$来表示。

总而言之，Emission和In-scattering会对当前传播路径贡献光子，Absorption和Out-scattering则会消耗当前路径光子，这里需要引入一些定义：

消亡系数$\sigma_t=\sigma_s(Out-scattering)+\sigma_a$可以用来描述被消耗比率。

反照率$\rho=\frac{\sigma_s}{\sigma_s+\sigma_a}=\frac{\sigma_s}{\sigma_t}$

辐射传输函数$L(x,v)$定义在空间中点$x$方向为$v$的radiance。


