# UETick 

## 初始化GraphTask

- void QueueAllTicks() 遍历所有TickFunction
- TickFunction->QueueTickFunction(TTS, Context); 递归的先为Prerequisites QueueTickFunction(TTS, Context)

- 如果没有Prerequisites,实际上是TTS.QueueTickTask(&TaskPrerequisites, this, TickContext);
- StartTickTask : TickFunction->InternalData->TaskPointer = TGraphTask<FTickFunctionTask>::CreateTask(Prerequisites, TickContext.Thread).ConstructAndHold(TickFunction,...)
- AddTickTaskCompletion : TickTasks[StartTickGroup][EndTickGroup].Add(Task);

注意构造GraphTask时将计数器设置为传入的Prerequisites的数量+1.

```cpp
//TaskGraphInterfaces.h FBaseGraphTask
FBaseGraphTask(int32 InNumberOfPrerequistitesOutstanding)
	: ThreadToExecuteOn(ENamedThreads::AnyThread)
	, NumberOfPrerequistitesOutstanding(InNumberOfPrerequistitesOutstanding + 1)
	{
		...
	}
```

这里应用的是ConstructAndHold,需要手动调用Unlock将这个额外的计数器减去.(如果是ConstructAndDispatchWhenReady,则会立刻减去这个计数器)

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

GraphTask处理Task依赖的方式是遍历所有Prerequisites,将当前的GraphTask加入到前置节点的SubsequentList中去,在一个GraphTask执行完毕后,会将SubsequentList中所有的的GraphTask计数器-1:

```cpp
//TaskGraphInterfaces.h TGraphTask
void ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread, bool bDeleteOnCompletion) override
{
	...
	TTask& Task = *(TTask*)&TaskStorage;
	{
		Task.DoTask(CurrentThread, Subsequents);
		Task.~TTask();
	}
	...
	if (TTask::GetSubsequentsMode() == ESubsequentsMode::TrackSubsequents)
	{
		Subsequents->DispatchSubsequents(NewTasks, CurrentThread);
	}
}

//TaskGraph.cpp
void FGraphEvent::DispatchSubsequents(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThreadIfKnown)
{
	...
	bool bWakeUpWorker = false;
	TArray<FBaseGraphTask*> PoppedTasks;
	SubsequentList.PopAllAndClose(PoppedTasks);
	for (int32 Index = PoppedTasks.Num() - 1; Index >= 0 ; Index--) // reverse the order since PopAll is implicitly backwards
	{
		FBaseGraphTask* NewTask = PoppedTasks[Index];
		NewTask->ConditionalQueueTask(CurrentThreadIfKnown, bWakeUpWorker);
	}	
}
```

如果计数器归零,就可以加入到Thread的执行队列中去了:
```cpp
//TaskGraphInterfaces.h FBaseGraphTask
void ConditionalQueueTask(ENamedThreads::Type CurrentThread, bool& bWakeUpWorker)
{
	if (NumberOfPrerequistitesOutstanding.Decrement()==0)
	{
		QueueTask(CurrentThread, bWakeUpWorker);
		bWakeUpWorker = true;
	}
}

//TaskGraphInterfaces.h FBaseGraphTask
void QueueTask(ENamedThreads::Type CurrentThreadIfKnown, bool bWakeUpWorker)
{
	TaskTrace::Scheduled(GetTraceId());
	//调用到FNamedTaskThread EnqueueFromThisThread
	FTaskGraphInterface::Get().QueueTask(this, bWakeUpWorker, ThreadToExecuteOn, CurrentThreadIfKnown);
}

//TaskGraph.cpp FNamedTaskThread 
virtual void EnqueueFromThisThread(int32 QueueIndex, FBaseGraphTask* Task) override
{
	uint32 PriIndex = ENamedThreads::GetTaskPriority(Task->GetThreadToExecuteOn()) ? 0 : 1;
	int32 ThreadToStart = Queue(QueueIndex).StallQueue.Push(Task, PriIndex);
}
```

AddTickTaskCompletion函数将Task加入到TickTasks[StartTickGroup][EndTickGroup]这个数组中,CompletionEvent则是加入到TickCompletionEvents[EndTickGroup]的数组中.

```cpp
//TickTaskManager.cpp
FORCEINLINE void AddTickTaskCompletion(ETickingGroup StartTickGroup, ETickingGroup EndTickGroup, TGraphTask<FTickFunctionTask>* Task, bool bHiPri)
{
	if (bHiPri)
	{
		HiPriTickTasks[StartTickGroup][EndTickGroup].Add(Task);
	}
	else
	{
		TickTasks[StartTickGroup][EndTickGroup].Add(Task);
	}
	new (TickCompletionEvents[EndTickGroup]) FGraphEventRef(Task->GetCompletionEvent());
}
```

## 执行Task

- 完成初始化后后面就是执行Task了,主要的方法是TickTaskSequencer.ReleaseTickGroup(Group, bBlockTillComplete);

- 这里分为两个主要部分:
- 1. DispatchTickGroup(ENamedThreads::GameThread, WorldTickGroup);
- 2. FTaskGraphInterface::Get().WaitUntilTasksComplete(TickCompletionEvents[Block], ENamedThreads::GameThread);

DispatchTickGroup对QueueAllTicks过程中添加到HiPriTickTasks和TickTasks中的Task进行Unlock,将Task加入到Thread的执行队列:
(这里也就对应了创建GraphTask时的ConstructAndHold,如果没有前置依赖,此处Unlock才会Task加入到Thread的执行队列中去)

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

//TaskGraphInterfaces.h FBaseGraphTask
void QueueTask(ENamedThreads::Type CurrentThreadIfKnown, bool bWakeUpWorker)
{
	TaskTrace::Scheduled(GetTraceId());
	//调用到FNamedTaskThread EnqueueFromThisThread
	FTaskGraphInterface::Get().QueueTask(this, bWakeUpWorker, ThreadToExecuteOn, CurrentThreadIfKnown);
}

//TaskGraph.cpp FNamedTaskThread 
virtual void EnqueueFromThisThread(int32 QueueIndex, FBaseGraphTask* Task) override
{
	uint32 PriIndex = ENamedThreads::GetTaskPriority(Task->GetThreadToExecuteOn()) ? 0 : 1;
	int32 ThreadToStart = Queue(QueueIndex).StallQueue.Push(Task, PriIndex);
}
```

WaitUntilTasksComplete可以分成两个部分来理解:

```cpp
//TaskGraph.cpp
void WaitUntilTasksComplete(const FGraphEventArray& Tasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread) final override
{
	...
	// named thread process tasks while we wait
	TGraphTask<FReturnGraphTask>::CreateTask(&Tasks, CurrentThread).ConstructAndDispatchWhenReady(CurrentThread);
	ProcessThreadUntilRequestReturn(CurrentThread);
}

ProcessThreadUntilRequestReturn是真正执行Tick逻辑的部分,这里将会调用到FNamedTaskThread的ProcessTasksNamedThread,依次从队列中取出Task进行Execute.(也就是最终调用到Actor.Tick,ActorComponent.TickComponent)的地方.

```cpp
//TaskGraph.cpp FNamedTaskThread
uint64 ProcessTasksNamedThread(int32 QueueIndex, bool bAllowStall)
{
	...
	//会被后面的FReturnGraphTask中断!
	while (!Queue(QueueIndex).QuitForReturn)
	{
		FBaseGraphTask* Task = Queue(QueueIndex).StallQueue.Pop(0, bStallQueueAllowStall);
		if(!Task)
		{
			...
		}
		else
		{
			//这里执行Task,会调用真正的Tick逻辑,并对Subsequent的Task进行计数器递减,如果减到0最终也会加入到Queue(QueueIndex).StallQueue这个队列
			Task->Execute(NewTasks, ENamedThreads::Type(ThreadId | (QueueIndex << ENamedThreads::QueueIndexShift)), true);
		}
	}
	return ProcessedTasks;
}

//TaskGraphInterfaces.h TGraphTask
void ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread, bool bDeleteOnCompletion) override
{
	...
	TTask& Task = *(TTask*)&TaskStorage;
	{
		Task.DoTask(CurrentThread, Subsequents);
		Task.~TTask();
	}
	...
	if (TTask::GetSubsequentsMode() == ESubsequentsMode::TrackSubsequents)
	{
		Subsequents->DispatchSubsequents(NewTasks, CurrentThread);
	}
}

//TaskGraph.cpp
void FGraphEvent::DispatchSubsequents(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThreadIfKnown)
{
	...
	bool bWakeUpWorker = false;
	TArray<FBaseGraphTask*> PoppedTasks;
	SubsequentList.PopAllAndClose(PoppedTasks);
	for (int32 Index = PoppedTasks.Num() - 1; Index >= 0 ; Index--) // reverse the order since PopAll is implicitly backwards
	{
		FBaseGraphTask* NewTask = PoppedTasks[Index];
		NewTask->ConditionalQueueTask(CurrentThreadIfKnown, bWakeUpWorker);
	}	
}
```

FReturnGraphTask则是将必须在当前TickGroup执行完毕(或者在前面的TickGroup已经执行完毕)的GraphTask作为依赖Prerequisites创建一个ReturnGraphTask,这个Task的作用是当Prerequisites都执行完毕时,中断任务队列Queue的执行:

```cpp
//TaskGraphInterfaces.h
class FReturnGraphTask
{
	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		FTaskGraphInterface::Get().RequestReturn(ThreadToReturnFrom);
	}
}

//TaskGraph.cpp FTaskGraphCompatibilityImplementation
void RequestReturn(ENamedThreads::Type CurrentThread) final override
{
	int32 QueueIndex = ENamedThreads::GetQueueIndex(CurrentThread);
	CurrentThread = ENamedThreads::GetThreadIndex(CurrentThread);
	Thread(CurrentThread).RequestQuit(QueueIndex);
}

//TaskGraph.cpp FNamedTaskThread
virtual void RequestQuit(int32 QueueIndex) override
{
	Queue(QueueIndex).QuitForReturn = true;
}
```

