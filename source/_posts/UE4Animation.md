---
title: UE4Animation
date: 
categories:
- UE4
mathjax: true
---


## Animation BluePrint

AnimationBP类似于Unity中的Animator,主要功能就是用状态机的形式播放动画.

打开动画蓝图的编辑窗口后，左下角可以看到EventGraph，AnimationGraph,AnimationLayers,Functions,Macros,Variables,EventDispatchers几个部分。

### EventGraph

初始工程角色的动画蓝图中EventGraph每帧都会从Pawn的Movement组件中获取角色移动状态，并根据这些状态的数值来设置动画蓝图中的预设变量(Variables).可以理解为根据游戏逻辑设置动画蓝图的参数，AnimationGraph中的状态机就会根据这些变量选取合适的动画进行播放.


### AnimationGraph

最简单的AnimationGraph，一个状态机决定最终播放的动画

![](UE4Animation/AnimBP_AnimationGraph_General.png)


展开状态机,可以看到状态机中状态的切换规则

![](UE4Animation/AnimBP_AnimationGraph_StateMachine.png)

每个State都会输出一个动画,如何输出也可以自定义蓝图来实现.

Idle/Run这个状态的输出动画,就是由几个动画根据参数进行Blend来决定的:


#### Anim Montage Slot

AnimationGraph中可以看到状态机和输出节点之间还有一个名为Slot'DefaultSlot'的节点,这其实就是Anim Montage Slot.

这个节点可以在需要的时候用指定动画覆盖掉Source的动画,可以这样理解:

```
if Slot'DefaultSlot' Playing Animation then
    Outpose = Slot'DefaultSlot' Animation
else 
    Outpose = Source Animation
```

个人理解动画蒙太奇是十分适合用于技能动画的实现.通常来说技能动画会是一长串全身所有骨骼都会参与的序列,播放技能时直接覆盖常驻状态机的动画十分合理,并且,状态机只需要考虑常驻状态的切换,不需考虑技能动画,逻辑隔离十分清晰.(如果用Unity的Animator,就只能在Animator状态机中加技能动画状态,使用变量切换来播放动画了)

后面实现ACT游戏中常见的搓招,连击也就是通过动画蒙太奇来实现)
