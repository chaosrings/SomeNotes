# UE 属性同步


## 

数据下发Handle+真正的数据,

找到对应的Handle的RepLayoutCmd,用RepLayoutCmd中的Offset找到Property在UObject中的内存偏移,从InBunch中读取反序列化到对应的内存地址中.