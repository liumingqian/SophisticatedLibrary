## ISM

- ISM的所有实例一起，无论如何都只使用1个drawcall
- ISM的LOD是以所有实例的包围盒计算的，统一变化，逐实例变换在世界坐标变换后被应用
- ISM没有自己的剔除逻辑
- 每个实例网格都有lightmass生成的lightmap或shadowmap，并将在预计算中平铺在一起，因此光照贴图分辨率的设置必须很小，以便所有贴图平铺在一起不会超过最大纹理分辨率（4096）
- Instance物体的轴心位置通过将（0，0，0）变换到世界坐标系得到，如ObjectPosition等节点应该在顶点着色器中使用，可以通过将ObjectPosition传入CustomUV，或添加VertexInterpolater实现。

## HISM

- HISM的剔除不在常规的可见性计算流程中，而是在收集动态MeshBatch流程中完成，不支持软光栅和cull distance volume剔除

- HISMcomponent和他的父组件缩放始终为1.0，非等比缩放会引起剔除问题

  HISM(Hierarchical Instanced Static Mesh Component)继承自ISM，与ISM不同的是基于分层实现，分层结构是一个KD-Tree。ISM的全部模型都是用同样的LOD层级，HISM虽然开销大些，但可以为不同的实例使用不同的LOD层级，并分区域进行剪裁和LOD计算。UE中植被就是HISM的子类。HISM的实现有两部分，一部分是构建分层（自动化的，每次修改它的instance个数、位置等都会触发）并保存到文件中，另一部分则是基于分层的可见性计算、LOD和渲染组织。

  

  在生成MeshBatch时，从ClusterTree根节点开始计算节点对应的包围盒是否在视锥体内，如果在则继续查询子节点。将可见节点对应的首尾instance id记录到数组中（必要时合并连续区间），得到多段首尾instance id。

  在HISM的树中，每个节点叫FClusterNode，除了BoundingBox信息外，FClusterNode并不以数组等形式保存子节点或者instance数据，节点只保存当前节点的First和last instance序号还有first和last的子节点编号，这么做的原因是因为HISM在绘制的时候如果有剔除结果的变化的时候，不会去重组instancebuffer。所以只能修改DrawIndexedInstanced的StartIndex和EndIndex来实现动态的绘制。所以在构建时每个节点里面的instance序号都是连续的。

  并不是每一层的节点都会参与到剔除里面，UE4会根据一系列参数计算确定每层节点是否参与剔除。在每帧的绘制中，UE4的渲染器都会在Visibility的计算过程中调用FHierarchicalStaticMeshSceneProxy::GetDynamicMeshElements 根据剔除的结果算出需要扔到GPU的节点，平衡剔除的粒度和剔除的效果，是个很复杂的事情，HISM虽然是静态场景，但是每帧都要更新，如果数据结构太复杂，或者剔除的粒度太细（比如每个instance都画一个box去剔除），那么每帧用在更新数据上的时间对于CPU来说也是很大的负担，另外，粒度太细，instance减少dc的作用就没了，极端情况下，每个instance一个节点的话，效率和不做instance直接绘制是一样的。反之，如果剔除粒度太粗，那么GPU压力就会迅速上升，因为大部分的东西剔除不掉。

  > 影响树的构建的参数：
  >
  > - MinInstancesPerOcclusionQuery：最小的一次occlusionquery一起查询的instance数目，会影响最后build出树的结构。
  >
  > - MaxOcclusionQueriesPerComponent：最大的每个HISMcomponent 执行query的DrawCall 数目，这个可以控制你的游戏用在剔除的DrawCall数量的上限。
  >
  > - Min Verts toSplit：分割节点的顶点数，这只是一个建议值，ue4默认是1024，也就是只有大于1024个顶点，才会继续向下分割节点构造树。
  >
  > - InternalBranchingFactor：分支因子，一般最小肯定是2，不过如果是1可以认为是强行按照每个instance剔除。
  >
  > 以上参数得到两条公式：
  >
  > - occlusionLayerTarget（instance总数/InstancesPerOcclusionQuery） * InstancesPerOcclusionQuery（每个遮挡查询覆盖的Instance数） =  OcclusionQueriesPerComponent(每个hism component的遮挡查询数）
  >
  > - Lod0顶点数 * InstancesPerNode（PerLeaf)（每个节点可以有多少个instance) = VertsToSplit

## Dynamic Instance

就是动态判断绘制的物体材质，Vertex Buffer（mesh+pipeline state）等是否一样，一样的物体合并到一起绘制。

因为FMeshDrawCommands捕获了绘制在RHI级别上所需的所有状态，因此可以将具有相同着色器i绑定的绘制调用合并到实例化绘制中。合并到实例化绘制的两个DrawCall只有着色器中的instanceID会变化。可以合并的内容：

使用相同材质实例的DrawCall、使用相同光照贴图资源的DrawCall，使用相同UStaticMesh的DrawCall。

Dynamic Instance实现的核心是UE4实现了一个GPU Scene——用一个Buffer来存储全场景每个Primitive的Transform等信息，这样在组装出Instance Buffer之后只需用PrimitiveID就可以访问到自己的位置相关数据，从而实现Instance渲染了。