#### 引用资源

- 直接属性引用：通过UPROPERTY公开

- 构造时引用：在构造函数中通过ConsructorHelper::FObjectFinder来加载某个确定路径下的资源。对象加载并实例化的时候会加载以这种硬性方式引用的资源，需要考虑内存会因为同时加载许多资源而迅速增加。

- 间接属性引用：UPROPERTY使用TSoftObjectPtr，LazyLoad，使用资源时手动加载。使用时首先用 IsPending() 方法判断资源是否已准备好可供访问，没有则通过LoadObject<>() 方法、StaticLoadObject() 或 FStreamingManager 来加载对象（前两个方法是同步加载，可能导致帧率突增）。

- 运行时查找加载：如果UObejct已经加载或已经创建，使用FindObject，如果对象未加载，使用LoadObject。注意LoadObejct内部执行的操作于FindObject，因此不用先Find再Load

  **Reference**

  [引用资源](https://docs.unrealengine.com/4.26/zh-CN/ProgrammingAndScripting/ProgrammingWithCPP/Assets/ReferencingAssets/)

### 性能优化

- 