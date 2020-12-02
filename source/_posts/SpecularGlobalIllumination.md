---
title: SpecularGlobalIllumination
date: 
categories:
- Global Illumination
tags:
---

镜面材质的全局光照实现方式大致也可以分为预计算和近似计算两种实现方式。

## Localized Environment Maps

在Unity被称作Reflection Probe。

这种实现方式的原理与局部光照中的Specular IBL十分类似，主要有两点不同：

1.Specular IBL采样radiance的贴图是全局的EnvMap，LocalizedEnvMap则是在指定空间区域（多为立方体）内收集周围的radiance信息，在这个区域内的镜面材质物体着色才会使用这个LocalizedEnvMap。

![localizedEnvMap](SpecularGlobalIllumination/LocalizedEnvMap.png)


2.Specular IBL中环境光被认为是从无穷远点入射到表面，因此直接使用视线反射向量来采样环境贴图并不会带来误差——相对于无穷远处，场景中的任意物体都可以视作在环境贴图的中心。但LocalizedEnvMap并不能这样处理，除非着色点正好位于作用区域的中心位置，越是偏离中心位置的着色点直接使用视线反射向量来采样LocalizedEnvMap产生的误差就会越大。

![reflectionProxy](SpecularGlobalIllumination/ReflectionProxy.png)

采样LocalizedEnvMap向量的计算方式如右图，首先根据视线向量$v$以及着色点法线$n$计算反射向量$r$，计算$r$与作用区域几何体的交点$p$，最终的采样向量$r`$为作用区域中心到交点$p$的向量。

除此之外，在Specular IBL中一些预处理的方法都可以用在LocalizedEnvMap，比如MipMap建立不同粗糙度的贴图，split-integral approximation等...

## Planar Reflections

![planarReflection](SpecularGlobalIllumination/PlanarReflection.png)

当要渲染的物体是一个平面时，可以通过直接渲染场景中其他物体与此平面的镜像来模拟镜面材质，这种方法十分简单易于理解，不仅能渲染完美镜面的反射效果，也能模拟带有一定粗糙度的光泽表面。缺陷在于消耗很大，相当于场景中的物体要被渲染两次。

## Screen Space Methods

这里就不得不提ScreenSpaceReflection，近年玩过的游戏大部分都有这个效果，渲染出来的反射效果看起来确实不错。