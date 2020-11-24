---
title: IrradianceEnvironmentMapping
date: 2020-11-24 23:18:19
categories:
- Local Illumination
tags:
---

EnvMap不仅能用于光泽反射表面的着色，也可以对漫反射表面着色。相比用来渲染光泽表面的EnvMap存储radiance数值，通过视线反射向量来索引取值，用来渲染漫反射表面的EnvMap存储的是irradiance数值，通过着色点的法线来索引取值。

为什么要存储irradiance而不是像Specular IBL那样存储radiance呢？可以回顾一下理想漫反射表面的渲染方程：

$L_o=\int_\Omega f(l,v)L_i (n·l)dl=\frac{c_{albedo}}{\pi}\int L_i(n·l)dl=\frac{c_{albedo}}{\pi}E$

在渲染时我们只需要irradiance和表面的反照率属性就能通过简单的乘法计算得到最终的着色结果。

预计算这个Irradiance EnvMap的过程就是一个数值积分，伪码如下：

```CG

```

![calculate](IrradianceEnvironmentMapping/CalculateIrradianceEnvMap.png)

