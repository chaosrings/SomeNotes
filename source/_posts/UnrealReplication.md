# 虚幻Replication想到哪写到哪 

## 客户端从DS同步Actor过程


NetConnection:ReceivedPacket,判断是否有已有Channel,没有的话需要创建

```
Channel = CreateChannelByName( Bunch.ChName, EChannelCreateFlags::None, Bunch.ChIndex );
```

创建后使用此Channel读取和处理Bunch.

```
Channel->ReceivedRawBunch(Bunch, bLocalSkipAck);
```

如果Bunch中有NetGUID相关数据,首先会处理NetGUID,创建NetGUID到CachedObject的Map:
```
void UPackageMapClient::ReceiveNetGUIDBunch( FInBunch &InBunch )
{
    ...
    while( NumGUIDsRead < NumGUIDsInBunch )
    {
        const FNetworkGUID LoadedGUID = InternalLoadObject( InBunch, Obj, 0 );
        NumGUIDsRead++;
    }
}

//传入参数都是从Bunch中读取
void FNetGUIDCache::RegisterNetGUIDFromPath_Client( const FNetworkGUID& NetGUID, const FString& PathName, const FNetworkGUID& OuterGUID, const uint32 NetworkChecksum, const bool bNoLoad, const bool bIgnoreWhenMissing )
{
    ...
    FNetGuidCacheObject CacheObject;
    CacheObject.PathName			= FName( *PathName );
    CacheObject.OuterGUID			= OuterGUID;
    CacheObject.NetworkChecksum		= NetworkChecksum;
    CacheObject.bNoLoad				= bNoLoad;
    CacheObject.bIgnoreWhenMissing	= bIgnoreWhenMissing;
    RegisterNetGUID_Internal( NetGUID, CacheObject );
}

FNetworkGUID UPackageMapClient::InternalLoadObject( FArchive & Ar, UObject *& Object, const int32 InternalLoadObjectRecursionCount )
{
    ...
    //注册CacheObject
    GuidCache->RegisterNetGUIDFromPath_Client( NetGUID, ObjectName, OuterGUID, NetworkChecksum, ExportFlags.bNoLoad, bIgnoreWhenMissing );
    //根据CacheObject中注册的信息获取Object
    Object = GuidCache->GetObjectFromNetGUID( NetGUID, GuidCache->IsExportingNetGUIDBunch );
    ...
}

UObject* FNetGUIDCache::GetObjectFromNetGUID( const FNetworkGUID& NetGUID, const bool bIgnoreMustBeMapped )
{
    UObject* Object = CacheObjectPtr->Object.Get();
    if ( Object != NULL )
    {
        return Object;
    }
    // See if this object is in memory
    Object = FindObjectFast<UObject>(ObjOuter, CacheObjectPtr->PathName);
    //Load From package
    ...
    CacheObjectPtr->Object = Object;
    return Object;
}
```
GetObjectFromNetGUID会依次查找UPackageMapClient.ObjectLookup,全局UObject,磁盘加载,来找到对应的UObject.
UPackage,CDO这种静态资源(NetGUID.IsStatic()),GetObjectFromNetGUID会直接得到对应的UObject.
如果是动态创建的,那么就需要后续的SerializeNewActor,SerializeObject才会赋值.


整批Bunch都读到可以提交时,会判断ActorChannel对应的Actor是否存在,如果不存在需要SerializeNewActor

```
UActorChannel::ProcessBunch(FInBunch & Bunch)
{
    AActor* NewChannelActor = NULL;
    bSpawnedNewActor = Connection->PackageMap->SerializeNewActor(Bunch, this, NewChannelActor);
}
```


SerializeNewActor会从Bunch中依次读取Actor的NetGUID,ArchetypeNetGUID,Location,Scale,Velocity,从World中Spawn后再注册到GuidCache中:

```
//Ar就是InBunch
bool UPackageMapClient::SerializeNewActor(FArchive& Ar, class UActorChannel *Channel, class AActor*& Actor)
{
    FNetworkGUID NetGUID;
    UObject *NewObj = Actor;
    SerializeObject(Ar, AActor::StaticClass(), NewObj, &NetGUID);
    if ( NetGUID.IsDynamic() )
    {
        UObject* Archetype = nullptr;
        UObject* ActorLevel = nullptr;
        FNetworkGUID ArchetypeNetGUID;
        SerializeObject(Ar, UObject::StaticClass(), Archetype, &ArchetypeNetGUID);
        ConditionallySerializeQuantizedVector(Location, FVector::ZeroVector, GbQuantizeActorLocationOnSpawn, bSerializeLocation);
        ConditionallySerializeQuantizedVector(Scale, FVector::OneVector, GbQuantizeActorScaleOnSpawn, bSerializeScale);
        ConditionallySerializeQuantizedVector(Velocity, FVector::ZeroVector, GbQuantizeActorVelocityOnSpawn, bSerializeVelocity);
        // Spawn actor if necessary (we may have already found it if it was dormant)
        if ( Actor == NULL )
        {
	        if ( Archetype )
	        {
                ...
                Actor = World->SpawnActorAbsolute(Archetype->GetClass(), FTransform(Rotation, SpawnLocation), SpawnInfo);
                GuidCache->RegisterNetGUID_Client(NetGUID, Actor);
            }
        }
    }
}
```

动态生成的NetGUID会在RegisterNetGUID_Client最终将Object指针赋值.

```
void FNetGUIDCache::RegisterNetGUID_Client( const FNetworkGUID& NetGUID, const UObject* Object )
{
    ...
    FNetGuidCacheObject CacheObject;
    CacheObject.Object = MakeWeakObjectPtr(const_cast<UObject*>(Object));
    RegisterNetGUID_Internal( NetGUID, CacheObject );
}
```



