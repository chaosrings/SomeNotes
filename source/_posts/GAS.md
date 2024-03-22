

# Attribute

AttributeSet定义在AbilitySystemComponent的Owner上,```void UAbilitySystemComponent::InitializeComponent()```会遍历找到AttributeSet,添加引用,后面修改就是找到对应的Attribute进行ApplyMod

同步时使用PlayerState的ActorChannel,为什么SubObjects可以在ActorChannel有Replicator呢?

查看
```
 	UnrealEditor-GameplayAbilities.dll!UAbilitySystemComponent::ReadyForReplication() 行 247	C++
 	UnrealEditor-Engine.dll!AActor::BuildReplicatedComponentsInfo() 行 767	C++
 	UnrealEditor-Engine.dll!AActor::DispatchBeginPlay(bool bFromLevelStreaming) 行 4110	C++
>	UnrealEditor-Engine.dll!AActor::PostActorConstruction() 行 3901	C++

```

Actor调用Component的ReadyForReplication
```cpp
void AActor::BuildReplicatedComponentsInfo()
{
	checkf(HasActorBegunPlay() == false, TEXT("BuildReplicatedComponentsInfo can only be called before BeginPlay."));

	for (UActorComponent* ReplicatedComponent : ReplicatedComponents)
	{
		// Ask the actor if they want to override the replicated component
		const ELifetimeCondition NetCondition = AllowActorComponentToReplicate(ReplicatedComponent);

		const int32 Index = ReplicatedComponentsInfo.AddUnique(UE::Net::FReplicatedComponentInfo(ReplicatedComponent));
		ReplicatedComponentsInfo[Index].NetCondition = NetCondition;

		if (!ReplicatedComponent->IsReadyForReplication())
		{
			ReplicatedComponent->ReadyForReplication();
		}
	}
}
```

AbilitySystemComponent注册SubObjects:
```cpp
void UAbilitySystemComponent::ReadyForReplication()
{
	Super::ReadyForReplication();
	if (IsUsingRegisteredSubObjectList())
	{
		for (UGameplayAbility* ReplicatedAbility : GetReplicatedInstancedAbilities_Mutable())
		{
			if (ReplicatedAbility)
			{
				AddReplicatedSubObject(ReplicatedAbility);
			}
		}

		for (UAttributeSet* ReplicatedAttribute : SpawnedAttributes)
		{
			if (ReplicatedAttribute)
			{
				AddReplicatedSubObject(ReplicatedAttribute);
			}
		}
	}
}
```

# AbilityTask

一个异步过程,ASC继承自UGameplayTasksComponent,其中维护TickingTasks,SimulatedTasks.其中SimulatedTasks会同步给Simulate端.(比如UAbilityTask_ApplyRootMotionConstantForce)

UAbilityTask_PlayMontageAndWait播放Montage,LocalPredict如果被拒绝会停止播放蒙太奇.

```cpp
float UAbilitySystemComponent::PlayMontage(...)
{
	...
	FPredictionKey PredictionKey = GetPredictionKeyForNewAction();
	PredictionKey.NewRejectedDelegate().BindUObject(this, &UAbilitySystemComponent::OnPredictiveMontageRejected, NewAnimMontage);
}


void UAbilitySystemComponent::OnPredictiveMontageRejected(UAnimMontage* PredictiveMontage)
{
	...
	UAnimInstance* AnimInstance = AbilityActorInfo.IsValid() ? AbilityActorInfo->GetAnimInstance() : nullptr;
	if (AnimInstance && PredictiveMontage)
	{
		// If this montage is still playing: kill it
		if (AnimInstance->Montage_IsPlaying(PredictiveMontage))
		{
			AnimInstance->Montage_Stop(MONTAGE_PREDICTION_REJECT_FADETIME, PredictiveMontage);
		}
	}
}
``````