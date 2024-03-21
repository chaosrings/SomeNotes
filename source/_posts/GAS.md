

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