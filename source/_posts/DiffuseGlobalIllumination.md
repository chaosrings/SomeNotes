---
title: DiffuseGlobalIllumination
date: 
categories:
- Global Illumination
tags:
---

全局光照除了处理遮挡关系的AmbientOcclusion之外，还有一个关键在于物体间的间接光照，回想局部光照模型中的光线路径会发现，无论是Diffuse还是Specular部分都可以用$L(D|S)E$来描述(L:光源 D:漫反射 S:镜面反射 E:眼)，也就是到达人眼时，光线仅经过了一次弹射。但实际上物体之间是会有间接光照的，真正的光线路径应该是$L(D|S)^*E$。实时渲染中并不能像光线追踪那般递归计算光线路径(消耗太大，光栅化架构也不合适)，因此实时渲染中的全局光照多用预计算和高效近似计算两种方式来实现。

## Surface Prelighting

简单来说就是将场景中的物体分为static和dynamic两类，static物体与光源以及其余static物体会在实时渲染之前使用离线渲染的算法预先计算好正确的光照交互的结果（通常是缓存Irrdiance），实时渲染时只需要取出结果与表面材质进行计算。dynamic物体则还是使用LocalIllumination的模型进行光照计算，也可以使用一些近似算法来得到不那么PhysicallyBased的光照结果。

Unity的LightMap原理应该就是这样,可以参考shader中对LightMap的使用：

```CG
float4 albedo=tex2D(_MainTex,i.uv);
half3 lm = DecodeLightmap(UNITY_SAMPLE_TEX2D(unity_Lightmap, i.uv1));
float3 diffuse = albedo.rgb * lm;
```

不过预先计算表面Irrdiance的方式是有缺陷的，在预计算时就需要得知法线的方向，无法在运行时通过法线贴图的方式得到更为精细的表面细节，因此又引申出了Directional Surface Prelighting

### Directional Surface PreLighting

Directional Surface PreLighting的思路很简单，既然一个既定方向的Irrdiance无法满足要求，那么可以把所有方向的Irrdiance都预计算好，实时渲染时通过表面法线向量来索引取值。实际上这也是最常用的做法，预计算所有方向的Irrdiance并投影到球谐函数，在Texture中存储所有球谐系数，实时渲染时通过球谐系数来重建光照信息，传入法线来获得Irrdiance参与光照计算。通常而言投影到3阶球谐系数就能得到不错的结果，不过3阶系数需要9x3=27个float量，是简单Irrdiance的9倍消耗，因为这个原因也有一些优化的算法。

### Volumeterically PreLighting

既然我们能将对静态物体的光照预计算的结果存储于光照贴图LightMap并在实时渲染中使用，那么是否可以更进一步预计算存储场景中一些指定点（自动生成或人工指定）的Irrdiance，实时渲染时用这些已知点的Irrdiance信息通过插值的方式来查询场景中任意一点的Irrdiance信息呢？实际上多数游戏引擎动态物体的GlobalDiffuseIllumination就是基于这个原理来实现的。

#### Irrdiance Volume


#### Unity tetrahedral Interpote