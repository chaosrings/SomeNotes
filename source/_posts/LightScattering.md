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

反照率定义为$\rho=\frac{\sigma_s}{\sigma_s+\sigma_a}=\frac{\sigma_s}{\sigma_t}$，反照率接近0时意味着光子几乎都是被介质吸收，像发电厂排放的黑烟就是这类介质，反照率为1意味着与介质接触的大部分光子都是被散射而不是被吸收，像云和大气就是这类介质。

在不考虑参与介质时，可以认为摄像机入射radiance等于与视线相交的最近表面的出射radiance，也就是$L_i(c,-v)=L_o(p,v)$，光线追踪便是基于这个原理从眼发射射线，最近相交表面的颜色也就是最终相机像素点的颜色。但如果考虑到参与介质，这个等式就不成立了，光线在参与介质中传播时与介质交互，radiance会发生变化，这个变化可以表述为：

$L_i(c,-v)=T_r(c,p)L_o(p,v)+\int_{0}^{||p-c||}T_r(c,c-vt)L_{scat}(c-vt,v)\sigma_sdt$

$T_r(a,b)=e^{-\sigma_td}$，$d$为a两点间的距离，这里是均匀介质才有$T_r(a,b)=e^{-\sigma_td}$，否则$T_r(a,b)=e^{-\int_a^b\sigma_t(x)d||x||}$，这个公式也被称作Beer-Lambert定理。