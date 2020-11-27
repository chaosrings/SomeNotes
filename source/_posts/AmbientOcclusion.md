---
title: AmbientOcclusion
date: 2020-11-28 00:25:04
categories:
tags:
---

环境光遮蔽的原理可以直接从渲染方程推导出来，对Lambertian表面，任意一个着色点p的渲染结果$L_o$是$\frac{c_{albedo}}{\pi}$和Irrdiance的乘积，若假设在此着色点p的所有入射方向radiance为一常量$L_A$，最简单的环境光就是这种形式，那么该点接受的Irrdiance就为

$E(p,n)=\int_{l \in \Omega}L_A (n·l)^+dl=\pi L_A$

积分结果十分简单，不过有一些缺陷，这里并未考虑到着色点会被本模型的面片以及场景中其他模型面片的遮挡，若简单的假设面片被遮挡的方向入射radiance是0（实际上不是，有间接光照），未被遮挡的方向为$L_A$,设可见函数$v(p,l)$表明p点的l入射方向是否被遮挡，遮挡为0反之为1，那么Irrdiance的积分就可以表示为：

$E(p,n)=L_A\int_{l \in \Omega}v(p,l)(n·l)^+dl$