# UE 属性同步

## RepLayoutCmd

Unreal为每个需要同步的类生成RepLayout,主要描述了所有需要同步的属性,RepParentCmd数组为最顶层的成员,RepLayoutCmd则是将属性完全展开扁平化后的描述.

- 如果需要同步的变量都是基础类型,那么RepParantCmd的数量与RepLayoutCmd的数量其实是一致的.
- 如果需要同步的变量有结构体,或者动态数组,那么RepParentCmd需要对这些变量生成变化RepLayoutCmd.

![](./UnrealStateSync/RepLayoutCmd.png)

## 同步时序

Actor同步后,Replicator进行属性同步

![](./UnrealStateSync/CallStack.png)

## 具体实现


### 简单Property

这里的简单Property指基础类型,或者Replicated结构体中的成员.

数据下发Handle+Property序列化后的结果

![](./UnrealStateSync/BunchData.png)

找到对应的Handle的RepLayoutCmd,用RepLayoutCmd中的Offset找到Property在UObject中的内存偏移,从InBunch中读取反序列化到对应的内存地址中.

### 动态数组

以简单测试的数组为例:

```
USTRUCT(BlueprintType)
struct FFireInfo
{
	GENERATED_USTRUCT_BODY()

	UPROPERTY()
	int32 Index;

	UPROPERTY()
	int32 Num;
};

class AUnrealCppTestCharacter : public ACharacter
{
    UPROPERTY(ReplicatedUsing = OnRep_FireInfo)
    TArray<FFireInfo> FireInfoArray;
}

```

在DS的RPC实现中增加数组第二项的计数:
```
void AUnrealCppTestCharacter::DoFire_Implementation()
{
    if (FireInfoArray.Num() > 1)
    {
  	    ++FireInfoArray[1].Num;
    }
}
```

单次变更了FireInfoArray[1].Num,客户端接受BunchData:

![](./UnrealStateSync/CallStack.png)

除去UEChracter默认的同步变量,RepParentCmds与Cmds中可以看到这个Property的描述:

![](./UnrealStateSync/MySyncParentCmd.png)

![](./UnrealStateSync/MySyncLayouyCmd.png)

以其中一次变更的下发数据为例:

![](./UnrealStateSync/BunchDataInstance.png)

接受Bunch后,先读取Handle:

```

bool FRepLayout::ReceiveProperties(
	UActorChannel* OwningChannel,
	UClass* InObjectClass,
	FReceivingRepState* RESTRICT RepState,
	UObject* Object,
	FNetBitReader& InBunch,
	bool& bOutHasUnmapped,
	bool& bOutGuidsChanged,
	const EReceivePropertiesFlags ReceiveFlags) const
{
    ...
    ReadPropertyHandle(Params);
    ReceiveProperties_r(Params, StackParams);
    ...
}
```
此处读取Handle=47,实际上Handle大部分情况都是Cmds数组的下标+1:
![](./UnrealStateSync/ReadHandle.png)

找到对应的Cmd,也就是Cmds[46],判断为动态数组,进入数组的读取逻辑.

```
static bool ReceiveProperties_r(FReceivePropertiesSharedParams& Params, FReceivePropertiesStackParams& StackParams)
{
    for (int32 CmdIndex = StackParams.CmdStart; CmdIndex < StackParams.CmdEnd; ++CmdIndex)
    {
        const FRepLayoutCmd& Cmd = Params.Cmds[CmdIndex];
        ++StackParams.CurrentHandle;
        if(StackParams.CurrentHandle ==  Params.ReadHandle)
        {
            if(ERepLayoutCmdType::DynamicArray == Cmd.Type)
            {
                //定义StackParams CmdIndex + 1到Cmd.EndCmd - 1其实也就是结构体的成员数量
                //这个例子中结构体有两个元素,所以是[47,49) 左闭右开
                //LocalHandle为4,其实意思就是LocalHandle=1x2+2,变更数组中index=1的元素中的第二项
                FReceivePropertiesStackParams ArrayStackParams{
    				nullptr,
    				nullptr,
    				nullptr,
    				CmdIndex + 1,
    				Cmd.EndCmd - 1,
    				StackParams.RepNotifies,
    				0 /*ArrayElementOffset*/,
    				0 /*CurrentHandle*/,
    				StackParams.bShadowDataCopied
                };
                //读取数组长度
                uint16 ArrayNum = 0;
                Params.Bunch << ArrayNum;
                //读取LocalHandle 这里是4 
                ReadPropertyHandle(Params);
                const int32 ObjectArrayNum = ObjectArray->Num();
                for (int32 i = 0; i < ObjectArrayNum; ++i)
                {
                    ...
                    //关键函数,对每个成员依次累加 ArrayStackParams.CurrentHandle直到等于4
    	            ReceiveProperties_r(Params, ArrayStackParams)
                    ...
                }
            }
            else
            {
                //将Bunch中的数据反序列化到UObject中对应的位置
                //Cmd.Property->NetSerializeItem(Bunch, Bunch.PackageMap, Data + SwappedCmd);
                ReceivePropertyHelper(...);
            }
        }
    }
}
```
