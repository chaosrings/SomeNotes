
## Animation Blueprint

![](ALSAnimationBlueprint/AnimBP.png)

ALS的动画蓝图部分逻辑划分十分清晰,从左到右被不同颜色的注释框圈出来的依次是基础动画->瞄准动画->IK->布娃娃系统.

![](ALSAnimationBlueprint/LayerBlending.png)

基础动画的输出由全身动画BaseLayer,上半身叠加动画OverLayLayer以及静止动作BasePoses三个部分的输出混合得到.

本文主要分析BaseLayer部分逻辑的实现,如下图:

![](ALSAnimationBlueprint/BaseLayer.png)

红色箭头表示状态机间的引用关系,也可以理解为箭头源是子状态机,指向目标是父状态机.

(N) XXX States是站立姿势的动画状态机,(CLF) XXX States则是蹲姿的动画状态机.两种状态机的实现逻辑十分相似,站姿状态下,角色会有冲刺,转身等细节叠加,会比蹲姿状态机更为复杂.

## LocomotionCycles

LocomotionCycles状态机实现了角色移动状态下的状态机.

![](ALSLocomotionCycles/ALS_N_WalkRun_F_BlendSpace.png) | ![](ALSLocomotionCycles/ALS_N_WalkRun_FL_BlendSpace.png)
---|---
|MoveFoward|MoveForwardLeft|

![](ALSLocomotionCycles/LocomotionCyclesState.png)

首先用StrideBlend(步幅),WalkRunBlend(走/跑)两个参数控制动画融合(BlendSpace2D)得到六方向移动循环动画.

![](ALSLocomotionCycles/CycleBlending.png)

(N) CycleBlending这部分则是将生成的六方向移动动画缓存,并将前向移动的动画与冲刺动画融合,得到最终的(N)FMovement用于Directional States状态机.

这里为什么是六方向移动循环动画而不是[前,后,左,右,左前,右前,左后,右后]八个方向的动画?实际上这里的LF(左前)移动动画是角色

## Directional States状态机

![](ALSLocomotionCycles/DirectionalStates.png)

### 状态构成

以MoveF为例:

![](ALSLocomotionCycles/MoveFState.png)

MoveF输出动画由FMovement,BMovement,LFMovement,RFMovement四个动画用多引脚动画融合而成,控制这四个动画融合的参数为VelocityBlend[F,B,L,R],计算过程位于ALS_AnimBP->Calculate Velocity Blend.

![](ALSLocomotionCycles/CalVelocityBlend.png)

首先由移动速度方向(MovementComponent.Velocity)与Character的旋转方向计算相对旋转方向LocRelativeVelocityDir,并归一化为RelativeDirection.

再根据根据相对旋转方向RelativeDirection的x,y值计算F(前),B(后),L(左),R(右).

VelocityBlend.F=Clamp(RelativeDirection.x,0,1)

VelocityBlend.B=ABS(Clamp(RelativeDirection.x,-1,0))

VelocityBlend.L=ABS(Clamp(RelativeDirection.y,-1,0))

VelocityBlend.R=Clamp(RelativeDirection.y,0,1)

考虑极端的正前,后,左,右移动,VelocityBlend变量以及MoveF这个状态的输出分别为:

VelocityBlend=(1,0,0,0),MoveFState=cachedPos'(N) F Movement'

VelocityBlend=(0,1,0,0),MoveFState=cachedPos'(N) B Movement'

VelocityBlend=(0,0,1,0),MoveFState=cachedPos'(N) LF Movement'

VelocityBlend=(0,0,0,1),MoveFState=cachedPos'(N) RF Movement'

其余的情况的输出则是其中两组动画(F,B)与(LF,RF)的融合.

其他状态的构成与MoveF类似,不过为了左前移动<->右前移动,左后移动<->右后移动的切换自然,左右四个状态(LF,LB,RF,RB)的动画融合有一些变化:

![](ALSLocomotionCycles/MoveLFState.png)

以MoveLF状态为例,MoveLF动画融合中VelocityBlend.R参数控制的动画是cachedPos '(N) RB Movement'而不是MoveF中的cachedPos '(N) RF Movement',这样设计的目的是从左前移动切换到右前移动时,状态机的切换路径不是MoveLF->MoveRF而是MoveLF->MoveRB->MoveRF.

若是直接从MoveLF->MoveRF切换,人物的胸口朝向会瞬间切换(面向左前瞬切到面向右前)显得不自然.而MoveLF->MoveRB的切换过程中,胸口的面向不会发生变化,MoveRB->MoveRF的切换则会有一个缓慢过渡的过程.

### 状态切换

![](ALSLocomotionCycles/DirectionalStates.png)

状态机中的每个状态间的Transition都与MovementDirection变量相关,例如图中绿色为MovementDirection==Forward,紫色为MovementDirection==Right...

MovementDirection的计算逻辑位于ALS_AnimBP->Calculate Movement Direction

(ps:AmingRotation的获取位于ALS_Base_CharacterBP->BPI Get Essential ,其实就是将ControlRotation赋值给AimingRotation)

![](ALSLocomotionCycles/CalMovementDirection.png)

在VelocityDirection模式下(Character面朝速度方向,多数第三人称视角自由旋转相机的移动模式可以归为此类)MovementDirection的计算结果永远为Forward.

在LookingDirection和Animing模式下(Character面朝相机方向,第三人称视角锁定敌人的模式可以归为此类)MovementDirection由移动速度方向(MovementComponent.Velocity)与相机旋转方向(ControlRotation)的夹角计算得到,可以简单理解为两者夹角在-70°~70°为Forward,70°~110°为Right,110°~-110°为Backward,-110°~-70°为Left.

![](ALSLocomotionCycles/CalMovementDirectionSimple.png)

详细的计算过程为

![](ALSLocomotionCycles/CalMovementDirectionDetail.png)

## LocomotionDetail

![](ALSLocomotionDetail/LocomotionDetail.png)

LocomotionDetail会引用Locomotion Cycles的输出,并进一步叠加细节动画:

(N) Walking和(N) Running会直接使用Locomotion Cycles输出的动画,(N)Run Start和(N) Walk->Run则是叠加了倾斜动画:

(N)RunStart状态中使用VelocityBlend(也用于Locomotion Cycles->Directional States)融合四个方向的倾斜动画并叠加在基础的移动动画上:

![](ALSLocomotionDetail/RunStartState.png)

![](ALSLocomotionDetail/ALS_N_LocoDetail_Accel_F.png)|![](ALSLocomotionDetail/ALS_N_LocoDetail_Accel_L.png)
---|---
|ALS_N_LocoDetail_Accel_F|ALS_N_LocoDetail_Accel_L|

### 状态切换

LocomotionDetail状态机中Transition多与Gait变量相关,实际上(N)Walking->(N)Running的过程就是Gait值由Walking切换到(Running,Sprinting).

Gait一个有三个枚举值(Walking,Running,Sprinting)的枚举类型,计算逻辑位于ALS_Base_CharacterBP->Update Character Movement:

![](ALSLocomotionDetail/CalGait.png)

Get Allowed Gait会计算当前允许的最大Gait(Walking->Running->Sprinting是依次增大的)

![](ALSLocomotionDetail/CalAllowedGait.png)

Desired Gait在PlayerInputGraph中根据输入设置:
![](ALSLocomotionDetail/PlayerInputGraph.png)

GetActualGait是实际计算动画蓝图中用到的Gait过程,主要的逻辑就是根据当前速度是否大于WalkSpeed,RunSpeed...来决定Gait类型.

## Locomotion States

到LocomotionDetail这一层的状态机已经完成了运动状态的动画,Locomotion States将会引用LocomotionDetail的输出,实现运动状态与静止状态的切换:

![](ALSLocomotionStates/LocomotionStates.png)

(N) Moving直接使用了Locomotion Detail的输出:

![](ALSLocomotionStates/NMoving.png)

(N) Not Moving则是播放静止动画:

![](ALSLocomotionStates/NNotMoving.png)

(N) Stop需要处理左脚右脚停止的逻辑,是一个较为复杂的状态机:

![](ALSLocomotionStates/NStop.png)

## Crouching Left Forward States

![](ALSCLFLocomotion/CLFStates.png)

(N) Locomotion Cycles -> (N)Locomotion Detail ->(N) Locomotion States实现了站立姿势下的动画状态机,(CLF) Locomotion Cycles-> (CLF) Locomotion States则是完成了蹲姿下的状态机.

蹲姿的运动状态循环与站姿的运动状态循环实现逻辑十分类似,不过蹲姿简化了站立运动的步幅(StrideBlend)与行走状态(WalkRunBlend),因此直接使用了原始的动画片段执行CycleBlending:

![](ALSCLFLocomotion/CLFLocomotionCycles.png)

CycleBlending部分的逻辑与站姿也大致相同,不过前向移动的动画是直接使用原始动画片段,站姿则是行走动画与冲刺动画融合.

![](ALSCLFLocomotion/CLFCycleBlending.png)|![](ALSCLFLocomotion/CLFDirectionalStates.png)
---|---



## Main Grounded States

Main Grounded States引用了(N) Locomotion States,(CLF) Locomotion States的输出,主要实现了角色站姿和蹲姿的切换:

![](ALSMainGroundedStates/MainGroundedStates.png)

(N) Standing直接使用(N) Locomotion States状态机的输出,(CLF) Crouching LF使用(CLF) Locomotion States的输出:

![](ALSMainGroundedStates/NStanding.png)| ![](ALSMainGroundedStates/CLFCrouching.png)
---|---
|(N) Standing|(CLF) Crouching Left|

(N)->(CLF) Transition播放站立到蹲下的动画,(CLF)->(N) Transition与之相反播放蹲下到恢复站立的动画:

![](ALSMainGroundedStates/NtoCLFTransition.png)

每个状态间的Transition几乎都与Stance变量相关,大致的逻辑为Stance为Crouching时站立切换到蹲下,状态依次切换[(N) Standing]->[(N)->(C)]->[(N)->(CLF) Transition]->[(CLF) Crouching LF],Stance为Standing时蹲下切换到站立.

![](ALSMainGroundedStates/StanceTransition.png)

Stance变量通过玩家输入直接修改数值:

![](ALSMainGroundedStates/CalStance.png)

 
## Main Movement States

Main Movement States是输出Base Layer最终动画的状态机,插件自带实现了角色的行走和跳跃的状态机,其他运动状态(攀爬,游泳)则需要在这个基础下额外实现:

![](ALSMainMovmentStates/MainmovementStates.png)

Grounded状态会直接使用Main Grounded States的输出动画:

![](ALSMainMovmentStates/GroundedState.png)

Jump状态由Jump States状态机的输出动画叠加空中倾斜动画,再融合落地动画得到最终输出动画:

![](ALSMainMovmentStates/JumpState.png)

Jump States状态机的构造就十分熟悉了,先播放起跳动画,再循环播放空中动画,不过ALS还区分实现了起跳时是左脚在前还是右脚在前:

![](ALSMainMovmentStates/JumpStateMachine.png)

Land状态用于播放从空中落地后没有进行运动的落地动画,会播放完整的落地动画再退出到GroundedState:

![](ALSMainMovmentStates/LandState.png)

LandMovement状态播放落地后进行了地面的运动的落地动画,需要将落地动画与Main Grounded States的输出进行融合:

![](ALSMainMovmentStates/LandMovementState.png)

状态间的Transition则是依靠变量Movement State:

![](ALSMainMovmentStates/MovementStateTransition.png)