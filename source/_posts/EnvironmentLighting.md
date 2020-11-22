---
title: EnvironmentLighting
date: 2020-11-17 23:39:03
categories: 
- light source
tags: 
- 环境光照
---

环境光与面积光方向光类似，也是一种直接光照模型。

## Ambient Light

Ambient Light是最简单的环境光模型，其radiance是一个常量数值，不会随着方向改变。当Ambient Light照射在漫反射表面时，得到的出射radiance $L_o$将会是一个常量值：

$L_o=\frac{\rho_{ss}}{\pi}L_A\int_{l \in \Omega}n·l dl=L_A*\rho_{ss}$

对任意BRDF的表面

$L_o=L_A\int_{l \in \Omega}f(l,v) n·l dl$

积分项$\int_{l \in \Omega}f(l,v) n·l dl$可定义为一个$R(l)$函数(directional-hemispherical reflectance)，实时渲染中常把此项简化为一个常数值$c_{amb}$，就有$L_o=c_{amb}L_A$，也就是代码中常见的:

```CG
L_ambient=Light_ambient*ambientColor
```

不过这里的着色是没有考虑物体间的相互遮挡的，因此一些凹陷处的颜色会比正确的结果要亮，为了解决这个问题也引申出了一系列计算遮蔽的方法。

## Radiance Varying Light

区别于常量radiance的ambient light，另一种环境光形式则是radiance会随着方向变化。这种光源的radiance不是一个简单的数值，而是一个关于方向的函数。这个函数通常没有解析解(可以设想如何用一个函数表达式来描述大气层在照亮地面时的radiance)。因此，要在渲染中应用这种环境光，需要使用数值解，或是用函数近似的方式来实现。

### Environment Mapping 

将一个球面radiance函数记录在Texture上的方式也就是EnvironmentMapping了，是实时渲染中最常见的环境光表示方式。EnvironmentMapping优势在于简单高效，并且可以表示任意高频的radiance。因此，在需要渲染光泽材质时，就可以用EnvironmentMapping来记录周围环境的radiance信息，着色时只需要在着色点计算视线基于着色点法线的反射向量，在贴图中采样就能得到入射的radiance，也就是：

$r=2(n·v)n-v$

$L_i(v)=Sampler(EnvMapping,r)$

Specular Image-Based Lighting就是基于此实现的。

### SphericalHarmonics 

另外一种方式就是将环境光照投影到球谐函数，对于比较低频的光照信息，可以通过存储少量球谐系数来近似目标函数(参见SphericalHarmonics XD)。这个方法的优势在于相比EnvironmentMapping，所需要的存储空间极少。