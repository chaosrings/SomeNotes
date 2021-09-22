---
title: UE4Animation
date: 
categories:
- UE4
mathjax: true
---

## Animation BluePrint

打开动画蓝图的编辑窗口后，左下角可以看到EventGraph，AnimationGraph,AnimationLayers,Functions,Macros,Variables,EventDispatchers几个部分。

### EventGraph

初始工程角色的动画蓝图中EventGraph每帧都会从Pawn的Movement组件中获取角色移动状态，并根据这些状态的数值来设置动画蓝图中的预设变量（Variables），可以理解为根据游戏逻辑设置动画蓝图的参数，驱动AnimationGraph播放正确的动画。

### AnimationGraph

最简单的AnimationGraph，一个StateMachine输出一个Output Pose，展开StateMachine可以看到其中的State和Transition定义，这里的状态机和Unity的Animator状态机挺像的，每个状态由动画片段或者BlendTree组成，状态之间的Transition则是由预设变量（Variables）的数值，或者动画本身的属性来定义。