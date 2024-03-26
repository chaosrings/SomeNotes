# UETick 

- void QueueAllTicks() 遍历所有TickFunction
- TickFunction->QueueTickFunction(TTS, Context); 递归的先为Prerequisites QueueTickFunction(TTS, Context)

- 如果没有Prerequisites,实际上是TTS.QueueTickTask(&TaskPrerequisites, this, TickContext);
- StartTickTask : TickFunction->InternalData->TaskPointer = TGraphTask<FTickFunctionTask>::CreateTask(Prerequisites, TickContext.Thread).ConstructAndHold(TickFunction,...)
- AddTickTaskCompletion : TickTasks[StartTickGroup][EndTickGroup].Add(Task);

这部分关键就是在为每个Prerequisites,执行AddSubsequent(this),将自身添加到SubsequentList中.

```cpp

//TaskGraphInterfaces.h TGraphTask
template<typename...T>
TGraphTask* ConstructAndHold(T&&... Args)
{
	new ((void *)&Owner->TaskStorage) TTask(Forward<T>(Args)...);
	return Owner->Hold(Prerequisites, CurrentThreadIfKnown);
}

//TaskGraphInterfaces.h TGraphTask
TGraphTask* Hold(const FGraphEventArray* Prerequisites = NULL, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
{
	SetupPrereqs(Prerequisites, CurrentThreadIfKnown, false);
	return this;
}

//TaskGraphInterfaces.h TGraphTask
void SetupPrereqs(const FGraphEventArray* Prerequisites, ENamedThreads::Type CurrentThreadIfKnown, bool bUnlock)
{
	TaskConstructed = true;
	TTask& Task = *(TTask*)&TaskStorage;
	SetThreadToExecuteOn(Task.GetDesiredThread());
	int32 AlreadyCompletedPrerequisites = 0;
	if (Prerequisites)
	{
		for (int32 Index = 0; Index < Prerequisites->Num(); Index++)
		{
			FGraphEvent* Prerequisite = (*Prerequisites)[Index];
			if (Prerequisite == nullptr || !Prerequisite->AddSubsequent(this))
			{
				AlreadyCompletedPrerequisites++;
			}
		}
	}
	PrerequisitesComplete(CurrentThreadIfKnown, AlreadyCompletedPrerequisites, bUnlock);
}
//TaskGraphInterfaces.h FGraphEvent
bool AddSubsequent(class FBaseGraphTask* Subsequent)
{
	bool bSucceeded = SubsequentList.PushIfNotClosed(Subsequent);
	return bSucceeded;
}

```

- 完成QueueAllTicks后,就是RunTickGroup了,调用到TickTaskSequencer.ReleaseTickGroup(Group, bBlockTillComplete);

- 这里分为两个主要部分:
- 1. DispatchTickGroup(ENamedThreads::GameThread, WorldTickGroup);
- 2. FTaskGraphInterface::Get().WaitUntilTasksComplete(TickCompletionEvents[Block], ENamedThreads::GameThread);

DispatchTickGroup对QueueAllTicks过程中添加到HiPriTickTasks和TickTasks中的Task进行Unlock.
```cpp
//TickTaskManager.cpp FTickTaskSequencer
void DispatchTickGroup(ENamedThreads::Type CurrentThread, ETickingGroup WorldTickGroup)
{
	for (int32 IndexInner = 0; IndexInner < TG_MAX; IndexInner++)
	{
		TArray<TGraphTask<FTickFunctionTask>*>& TickArray = HiPriTickTasks[WorldTickGroup][IndexInner]; //-V781
        for (int32 Index = 0; Index < TickArray.Num(); Index++)
        {
            TickArray[Index]->Unlock(CurrentThread);
        }
		TickArray.Reset();
	}
    //后面是对TickTasks进行同样的操作
}

//TaskGraphInterfaces.h TGraphTask
void Unlock(ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
{
    ...
	ConditionalQueueTask(CurrentThreadIfKnown, bWakeUpWorker);
}
```

ConditionalQueueTask实际上就是判断Task是否所有的前置Task已经执行完毕了.

最终会加入到GameThread.StallQueue中
```cpp
void ConditionalQueueTask(ENamedThreads::Type CurrentThread, bool& bWakeUpWorker)
{
	if (NumberOfPrerequistitesOutstanding.Decrement()==0)
	{
		QueueTask(CurrentThread, bWakeUpWorker);
		bWakeUpWorker = true;
	}
}
```