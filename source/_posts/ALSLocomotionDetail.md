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

LocomotionDetail状态机中Transition多与Gait变量相关,这是一个有三个枚举值(Walking,Runing,Sprinting)的枚举类型,计算逻辑位于ALS_Base_CharacterBP->Update Character Movement:

![](ALSLocomotionDetail/CalGait.png)

Get Allowed Gait会计算当前允许的最大Gait(Walking->Sprinting是依次增大的)

![](ALSLocomotionDetail/CalAllowedGait.png)

Desired Gait在PlayerInputGraph中根据输入设置:
![](ALSLocomotionDetail/PlayerInputGraph.png)

GetActualGait是实际计算动画蓝图中用到的Gait过程,主要的逻辑就是根据当前速度是否大于WalkSpeed,RunSpeed...来决定Gait类型.

## Locomotion States

到LocomotionDetail这一层已经完成了运动状态的动画,Locomotion States则是实现运动状态<->静止状态的切换:

![](LocomotionStates/LocomotionStates.png)

(N) Moving直接使用了Locomotion Detail的输出:

![](LocomotionStates/NMoving.png)

(N) Not Moving则是播放静止动画:

![](LocomotionStates/NNotMoving.png)

(N) Stop需要处理左脚右脚停止的逻辑,是一个较为复杂的状态机:

![](LocomotionStates/NStop.png)

## Main Grounded States

## Main movement States