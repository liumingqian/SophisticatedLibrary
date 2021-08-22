## Landscape Basic

一个Component有1或4个section，每个Section有2^n-1个quad(顶点数是2^n)，Quad是最小单元格，默认一个1米，除非改了Scale。地形以Section为单位进行LOD，如果Sectionsde ScreenSize差异不大，会自动合并Drawcall。Component是渲染单位，每个component有一个高度图纹理，因此顶点数必须是2的幂。相邻Component边缘的共享顶点会复制一份，因此考虑每个Component中的顶点数量是有意义的。较小的component大小可实现更快的LOD过渡，也可实现更多地形的遮挡，但是对于一个固定大小的地形，component越小，所需要的component越多，使用更少的component通常可以获得更好的性能。

Unreal的地形实现基于geometry mipmapping，每个地形块有自己的Weightmap和Heightmap，地形块间耦合度很小，只是需要获取4个邻居的lod值，传给vertex shader用于做缝合。Viwo的做法比较像Unity。

LandscapeLayerCoords 节点生成UV坐标，可用于将材质网络映射到地形地貌。

LandscapeVisibilityMask 节点移除地形的可见部分，以便在地形中创建洞进行如洞穴创建等操作。

Edit layers需要在Details面板打开Enable edit layers。

Unreal使用 -256 到 255.992 (cm)之间的值计算高度图的高度，并以 16 位精度存储，然后将该值乘以地形的Scale.Z

## 地形编辑

地形编辑器主要逻辑在LandscapeEdit.cpp。编辑时每个Component的数据存储在 ULandscapeComponent::PlatformData中。PlatformData是压缩的顶点数据，其中存储的顶点数据格式是FLandscapeMobileVertex，每个顶点对应一个FLandscapeMobileVertex：

```cpp
struct FLandscapeMobileVertex
{
	uint8 Position[4]; //x,y，所在Section在Component内的坐标（(SubX << 4) | SubY），挖洞的信息
	uint8 LODHeights[LANDSCAPE_MAX_ES_LOD_COMP*4];//0和1存储所有LOD高度中的最大最小值（抛弃掉了uint16的低八位），各个LOD的高度归一化到最大最小高度范围内，存储到LODHeights[2+Mip]中。
};
```

ULandscapeComponent::GeneratePlatformVertexData为Mobile生成了PlatformData，用HeightMapTexture的Mip获取各个LOD的高度数据，填充到FLandscapeMobileVertex数组里。

- ```
  MobileWeightmapTextures??
  ```

### 地形混合方式

- 基于权重：每个layer的颜色直接乘以该层权重值，然后再依次把所有算好的颜色值叠加一起
- 基于Height：在每层原始权重的基础上再叠加上一个来自与颜色图匹配的高度图的灰度值，后面会做个重新的归一化，这样就能保证输出的颜色亮度不会超过颜色纹理中的最高亮度。由于Weightmap分辨率只能与顶点数保持一致，这种混合方法比较能提升混合的细节。
- 基于Alpha：依照顺序逐层进行线性插值

编辑器中在对weightmap修改后，系统会做一个简单的判断，如果发现该层的权重值都变为零时，就把当前层移除掉。接着它会更新这个component的材质实例，根据当前的layer的使用情况生成对应的shader变体，假如有新的一层添加进来也会做同样的处理。

## 地形数据

总的来讲地形的信息都是编辑后保存时序列化到磁盘上，渲染时反序列化出来（PlatformData)使用。

PC使用顶点纹理来有效地存储高度和法线数据并计算 LOD 过渡，移动设备使用固定顶点和索引缓冲区。

地形块的上下左右邻居信息由全局变量SharedSceneProxyMap保存。

### VertexData

**SharedBuffer**

移动端地形运行时渲染数据由FLandscapeComponentSceneProxy::CreateRenderThreadResources()创建，填充在FLandscapeSharedBuffers里。每个Component有一个SharedBuffer，用SharedBuffersKey索引保存在全局变量SharedBuffersMap中。FLandscapeSharedBuffers中包含了

- Indexbuffer(每个LOD一个IndexBuffer)
- VertexBuffer
- LOD参数（以UniformBuffer的形式）。

FLandscapeSharedBuffers保存在一个全局变量SharedBuffersMap里，由SharedBuffersKey索引。

SharedBuffersKey只由SubsectionSizeQuads、NumSubsections唯一确定，因为属于一个Landscape的Component的SubsectionSizeQuads、NumSubsections是一模一样的，所以顶点分布（或者说顺序）也是一模一样的，所以没必要重复为每个Component生成不同的IndexBuffer，只要SubsectionSizeQuads、NumSubsections两个参数相同，就可以复用之前生成好的Index Buffer，因此所有Component的Proxy公用一个SharedBuffersKey。VertexBuffer是per proxy有自己的。

顶点排列根据有1个或4个sections，有两种情况：

<img src="Unreal Landscape.assets/vertex_order_section1x1.png" alt="Component内顶点的排列方式1x1" style="zoom:50%;" />

<img src="Unreal Landscape.assets/vertex_order_section2x2.png" alt="Component内顶点的排列方式2x2" style="zoom:50%;" />

**index buffer**

FLandscapeSharedBuffers::CreateIndexBuffers生成了IndexBuffer。ES3.1下根据Fortyth算法进行了[Post Transform Cache](https://www.khronos.org/opengl/wiki/Post_Transform_Cache)的优化，并且在LOD间共用了顶点以节省内存。

如果顶点数超过65535，则IndexBuffer的indexType是uint32，否则是uint16。

实际生成RHI资源发生在`FRawStaticIndexBuffer16or32<INDEX_TYPE>::InitRHI()`中，里面

- 创建了RHI Index buffer: `RHICreateIndexBuffer` -> `glBufferData`，这里的实现就是创建一个buffer对象来保存index
- 创建了SRV(Shader Resource View): `RHICreateShaderResourceView` -> `glGenTextures`、`glTexBuffer`/`glTexBufferRange`。

**vertex buffer**

FLandscapeComponentSceneProxy::CreateRenderThreadResources中创建了FLandscapeVertexFactoryMobile（如果使用VirtualTexture，会创建FLandscapeFixedGridVertexFactoryMobile）。ULandscapeComponent的PostLoad中，ULandscapeComponent::GeneratePlatformVertexData从Heightmap生成了PlatformData，以压缩形式保存。FLandscapeVertexFactoryMobile中的数据在CreateRenderThreadResources的时候用反序列化PlatformData得到的MobileRenderData中的VertexData填充。FLandscapeVertexFactoryMobile中最主要的数据是FDataType，FDataType在PC上的定义是

```cpp
struct FDataType
{
    /** The stream to read the vertex position from. */
    FVertexStreamComponent PositionComponent;
};
```

mobile上还多了LOD信息：

```
 /** stream which has heights of each LOD levels */
    TArray<FVertexStreamComponent,TFixedAllocator<LANDSCAPE_MAX_ES_LOD_COMP>> LODHeightsComponent;
```

这个数据结构是对应上面的FLandscapeMobileVertex，用PlatformData里面的数据填充的。

FVertexStreamComponent是一个保存了databuffer指针，以及data的offset和stride的数据结构。应该是在FLandscapeVertexFactoryMobile::InitRHI中被用于指定顶点属性偏移（会创建一个顶点格式声明列表：FVertexDeclarationElementList） 。

创建的LandscapeVertexFactory会保存在Proxy内，实际渲染时使用。

**uniform buffer**

LOD Uniform参数主要存在FLandscapeSectionLODUniformParameters里，对应shader中的LandscapeContinuousLODParameters。定义为：

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLandscapeSectionLODUniformParameters, )
	SHADER_PARAMETER(FIntPoint, Min)//Section的位置（Landscape本地Quad空间）
	SHADER_PARAMETER(FIntPoint, Size)//Section的大小（以Quad为单位）
	SHADER_PARAMETER_SRV(Buffer<float>, SectionLOD)//Section的LOD值
	SHADER_PARAMETER_SRV(Buffer<float>, SectionLODBias)//LOD偏移，未分析，略
	SHADER_PARAMETER_SRV(Buffer<float>, SectionTessellationFalloffC)//移动端不会用到Tessellation，略
	SHADER_PARAMETER_SRV(Buffer<float>, SectionTessellationFalloffK)//移动端不会用到Tessellation，略
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

这些Uniform参数每帧在FLandscapeRenderSystem::RecreateBuffers中向Render线程中的内存拷贝。

**OccluderMesh**

应该是生成了一个用于软光栅剔除其他Mesh的LODMesh。

在GeneratePlatformVertexData里用第一级Lod（具体使用哪级可以在ALandscape的Occluder Geometry LOD属性中配置）的顶点position填充了occluderVertices，应该也是序列化到PlatformData里，创建FLandscapeMobileRenderData的时候，从PlatformData里反序列化出来，保存到了MobileRenderData.OccluderVerticesSP。随后FLandscapeComponentSceneProxyMobile::CreateRenderThreadResources中，创建FLandscapeSharedBuffers的时候生成了OccluderVertices的indices信息，保存在FLandscapeSharedBuffers::OccluderIndicesSP中。

### Pixel Data

在PostEditChangeProperty中调用了ULandscapeComponent::GeneratePlatformPixelData，因为更换材质可能涉及WeightMap的重新排布。

**WeightMapTexture**

ULandscapeComponent::GeneratePlatformPixelData中创建了一张RGBA8格式的WeightMapTexture，从Heightmap的zw通道中拷贝了Normal信息放在WeightMapTexture的zw通道。根据当前Component Layer使用的情况填充WeightMap的通道，创建一张或多张WeightMap。如果挖洞，Visibility信息可能也会保存在WeightMap中。

**NormalMapTexture**

NormalMapTexture在PC上和Heightmap的值一样，在Shader里采样的是Heightmap的zw通道。Mobile上是MobileWeightmapTextures[0]。

**GIBakedBaseColorTexture**

开启Generate Mesh Distance Field和Generate Landscape Real-time GI Data的时候由系统生成，不能手动赋值。PerComponent一张图，分辨率非常低。



### 材质

可以添加Landscape Material Override，为每个LOD层级指定材质。
可以勾选UseDynamicMaterialInstance来给每个Component生成一个MID，可以创造踩雪等效果。
UpdateMaterialIstance_internal中实际给每个Component生成了一个ULandscapeMaterialInstanceConstant

**主要数据结构及其填充**

FLandscapeComponentSceneProxy中材质相关的主要数据结构是LODIndexToMaterialIndex（TMap<int,int>)和AvailableMaterials(材质数组）。

RenderThread中，FLandscapeComponentSceneProxy::GetStaticMeshElement中根据LODIndexToMaterialIndex数组索引了当前Lod需要的MaterialIndex。并以此为下标从AvailableMaterials中取值。

**LODIndexToMaterialIndex**通过ULandscapeComponent::UpdateMaterialInstances_Internal更新，所有可能用到的材质都会放在一起，放在MaterialInstances(PC)或MobileMaterialInterfaces（Mobile)中，通过LODIndexToMaterialIndex索引每一级LOD对应的在Material数组里的Index。LODIndexToMaterialIndex保存MaxLOD以下的所有LOD层级的MaterialIndex信息，有值的最后一级材质会自动往后面的LOD层级顺延。 FLandscapeComponentSceneProxy构造时由ULandscapeComponent的同名变量赋值得到。

**AvailableMaterials**也是在FLandscapeComponentSceneProxy构造函数中，从ULandscapeComponent的MobileMaterialInterfaces中取值。MobileMaterialInterfaces在ULandscapeComponent的GeneratePlatformPixelData中更新，如果面板上改动了材质，会通过PostEditChangeProperty最终调用到GeneratePlatformPixelData。

根据每块地形用的Layer数不同、本块地形上是否有洞，LOD层级等信息，ULandscapeComponent::GetCombinationMaterial会生成使用了不同的材质变体，保存在ALandscapeProxy::MaterialInstanceConstantMap中。地形材质中的Layer节点应该也是在这里构建和Material的联系。



## 地形渲染

加载场景调用FPrmitiveSceneInfo::AddStaticMeshes后调用了FLandscapeComponentSceneProxy::DrawStaticElements，从FirstLOD（可根据平台或硬件进行配置）到LastLOD分别调用FLandscapeComponentSceneProxy::GetStaticMeshElement生成MeshBatch。（相关Stat统计也发生在这里）

FLandscapeRenderSystem是一个工作在Render线程的数据结构，每个Component有一个，用FLandscapeKey作为索引保存在全局变量LandscapeRenderSystems（TMap）中。主要是被FLandscapePersistentViewUniformBufferExtension调用，更新Lod相关的UniformBuffer数据。FLandscapePersistentViewUniformBufferExtension是一个继承IPersistentViewUniformBufferExtension的UB扩展，在FMobileSceneRenderer中，会调用Scene->UniformBuffers.UpdateViewUniformBuffer，遍历所有UB扩展执行BeginRenderView函数，进行UB数据更新。FLandscapeRenderSystem在RecreateBuffer中将ComputeSectionPerViewParameters异步计算好的数据拷到UniformBuffer中。

```cpp
//主线程渲染循环
FMobileSceneRenderer::Render
FMobileSceneRenderer::InitViews
FPersistentUniformBuffers::UpdateViewUniformBuffer
FLandscapePersistentViewUniformBufferExtension::BeginRenderView
FLandscapeRenderSystem::BeginRenderView
FLandscapeRenderSystem::RecreateBuffers//更新UniformBuffer
```



### 顶点着色器

LandscapeVertexFactory.ush

```glsl
//CalcLOD方法根据这个顶点所在Section的LOD、邻接Section的LOD、顶点在Section的位置的重心坐标，算得了顶点的LOD（带小数）
float LODCalculated = CalcLOD(Intermediates.ComponentIndex, xy, Intermediates.InputPosition.zw);
float LodValue = floor(LODCalculated);//LOD 整数，范围[0, LastLOD]
float MorphAlpha = LODCalculated - LodValue;//LODCalculated的小数部分

//vertex x,y XY分布
float2 InputPositionLODAdjusted = floor(Vertex.xy / (2^LODValue)) / (SubsectionSizeVerts * (2^LODValue) - 1) * SubsectionSizeQuads;
float2 InputPositionNextLOD = floor( floor(Vertex.xy / (2^LODValue)) * 0.5 ) / (SubsectionSizeVerts*0.5f / (2^LODValue) - 1)  / SubsectionSizeQuads

//vertex z 高度
//height of this vertex at LOD of LODValue
float Height = lerp(MinHeight, MaxHeight, InputHeight);
//height of this vertex at LOD of (LODValue + 1)
float HeightNextLOD = lerp(MinHeight, MaxHeight, InputHeightNextLOD);

//混合XY分布和高度
vertexPosition = lerp(
            float3(InputPositionLODAdjusted, Height),
            float3(InputPositionNextLOD, HeightNextLOD),
            MorphAlpha);
```

CalcLOD方法以在Section内归一化的坐标xy，计算当前点向上下左右四个块的LOD的插值。以两条对角线将正方形的Section分割为4块的话，可以通过xy的值判断当前点属于哪一块，据此返回计算的LOD的值。

### LOD
lod数是SubsectionSizeQuads以2为底的对数，即如果15*15个Quads构成一个section，NumLod为log(2,16)=4。这个值作为MaxLod值，在FLandscapeComponentSceneProxy构造时就写好了.
LOD参数在ALandscape的Detail窗口中指定：

- LOD0ScreenSize:直观的看，近处的地块LOD的程度吧
- LOD0：直观的看，调整从近到远LOD变化的分界线的远近程度吧。。
- Other Lods:减少远处地形的顶点数（会影响到FractionalLOD)，直观地看，调整从近到远LOD变化分界线的软硬程度
- ComponentScreenSizeToUseSubSections:为了减少drawcall，Component离相机近的时候考虑把Subsection合并起来

移动平台应该是支持PLATFORM_SUPPORTS_LANDSCAPE_VISUAL_MESH_LOD_STREAMING的

FLandscapeRenderSystem中的SectionLODSettings为每个Section保存了一个LODSettingsComponent，主要是ForceLOD、LOD0和1的ScreenSize等信息，LODSettingsComponent是在CreateRenderThreadResource时，在FLandscapeRenderSystem::RegisterEntity中填充的。FLandscapeRenderSystem::ComputeSectionPerViewParameters（由FMobileSceneRenderer::InitViews异步调用）中根据SectionLODSettings、ScreenSize和LODScale计算FractionalLOD，最后将所有Section的LOD值作为一个数组存在CachedSectionLODValues中。最终保存到上面的Uniform Buffer中FLandscapeSectionLODUniformParameters::SectionLOD。在shader中需要获取相邻SectionLOD的数据，知道Section的坐标，也就能通过偏移拿到相邻Section的LOD了。

> 配置项
>
> - MobileLodBias：移动端的lod bias
> - MobileLandscapeLodBias：可以运行时根据特定设备执行额外偏差

### 剔除

有视锥剔除，开着软光栅的话也是能剃掉自己的（切换pc/es3.1，用freezerendering能看出来），将地形的Occluder Geometry Lod设为0以上就行

并不是预计算可视性volume能剃掉的东西

render setmesh draw time

CreateOccluderIndexBuffer

## 地形草

地形草的hism是在游戏运行时构建的，并不会序列化到资产文件里。LandscapeComponent::PostLoad时序列化出来的是GrassData(FLandscapeComponentGrassData)，只保存最基本的地形的HeightMapMip和Weightmap，并不保存Instance信息。weightMap帮助确定instance的位置，heightmap帮助instance对齐地形表面的高度和斜率。GrassData在保存地形编辑结果时(ALandscapeProxy::PreSave)，通过调用ALandscapeProxy::BuildGrassMaps生成了GrassData的HeightMip和WeightMap。

[Unreal Engine的地形草管理系统的缺陷分析报告](https://zhuanlan.zhihu.com/p/149771853)

### 草HISM生成

```cpp
//update grass调用堆栈
ULandscapeSubsystem::Tick
ALandscapeProxy::TickGrass
ALandscapeProxy::UpdateGrass
```

ULandscapeSubsystem的Tick遍历所有LandscapeProxy，并根据Landscape的Material里有没有Grass节点，来判断是否需要给Tick Grass。草的HISM由ALandscapeProxy::TickGrass调用ALandscapeProxy::UpdateGrass来更新，生成一个数据结构FCachedLandscapeFoliage::FGrassComp来保存草Hism相关信息，FCachedLandscapeFoliage::FGrassComp最终缓存在ALandscapeProxy的FoliageCache（类型是FCachedLandscapeFoliage，Transient）中。

UpdateGrass时首先对所有的LandscapeComponent，按包围盒到相机的距离排序，排除超出草的最大剔除距离的地形块。然后遍历所有GrassType的所有GrassVariety，为每种GrassVariety生成一个FCachedLandscapeFoliage::FGrassComp。FGrassComp中BasedOn landscapeComponent、GrassType、SubsectionX/Y、VarietyIndex等信息Hash为了FGrassCompKey，作为在ALandscapeProxy::FoliageCache中查找缓存的索引。如果有缓存就跳过Hism的Build过程。

如果没有缓存，创建新的UHierarchicalInstancedStaticMeshComponent和FAsyncGrassBuilder，创建一个FAsyncGrassTask，异步的调用FAsyncGrassBuilder::Build。由于构建时使用的随机种子是根据landscape component Hash出的，所以同个位置上的hism每次重新生成instance，都会出现在一模一样的地方。然后，遍历FAsyncGrassTask队列，检查Task如果完成了，则将FAsyncGrassBuilder中生成好的数据填充回HISMComponent。最后，根据GMinTimeToKeepGrass和GMinFramesToKeepGrass判断FoliageCache缓存的所有FGrassComp是否过期，并释放过期的HISMComponent。

FAsyncGrassBuilder用Halton算法或jitter方法（命令行可配置）生成GrassHISM。FAsyncGrassBuilder中保存草的权重信息的是GrassData（FLandscapeComponentGrassAccess）。对于Jitter方法（目前p6的方法），首先分配一个Instances数组（TArray<FInstanceLocal>），大小为SqrtMaxInstance*SqrtManInstance。相当于把地形分为SqrtMaxInstance*SqrtManInstance个格子，遍历每个格子位置，根据位置采样GrassData中的Height和Weight信息进行双线性插值，如果Weight和Random情况满足需求，则该Instance设置为有草的。随后第二次遍历Instances数组，随机旋转和Scale，生成InstanceTransform数组。将InstanceTransform数组提交给UHierarchicalInstancedStaticMeshComponent::BuildTreeAnyThread，进行HISM的构建。最终生成的草的数据会填充FStaticMeshInstanceData。并在ALandscapeProxy::UpdateGrass遍历FAsyncGrassTask队列时回填到HISM。

PIE模式下FLandscapeSharedBuffers构造时，会通过CreateGrassIndexBuffer创建一个草的IndexBuffer。FLandscapeComponentSceneProxy::CreateRenderThreadResources中也是将这个indexBuffer提交到GrassMeshBatch的。

ALandscapeProxy::UpdateGrass函数中从LandscapeComponent的MeshMapBuildData中取出地形的Lightmap和ShadowMap并赋给草的HISM的OverrideMapBuildData。

- GGrassDiscardDataOnLoad？

> 配置项
>
> - 可以通过命令行配置的变量见LandscapeGrass.cpp开头
> - GuardBand是草消失逐渐下去那条带吗
> - grass.CullSubsections 按SubSection剔除还是按Landscape Component剔除？

## 地形挖洞

- 材质里连Landscape VisibilityMask节点到OpacityMask

（实际上并不是

在调用FLandscapeComponentSceneProxy::GetStaticMeshElement生成地形MeshBatch的过程中，会调用FLandscapeComponentSceneProxyMobile::ApplyMeshElementModifier，移动端地形在此将MeshElement的模型数据替换成了`HoleData`所描述的含有洞的模型，而PC端是在shader里根据Visibility map，discard pixel来做的。

HoleData（FLandscapeMobileHoleData）是提前生成好序列化起来的，反序列化后保存在FLandscapeComponentSceneProxyMobile::MobileRenderData里。

## 问题

TexCoordOffsetParameter

- xyoffset和Tessellation都是SM4以上的功能

- LANDSCAPE_MAX_ES_LOD_COMP是啥？就是用于存储LOD信息的Vec4的个数

- 动态缓存DrawCommand?不是，只有选中地形时会通过FHitProxyMeshProcessor缓存DynamicDrawCommand

- 那mobile怎么LOD顶点数？顶点数据是固定的，每个LOD有一个IndexBuffer

- 地形哪里费：TRACE_CPUPROFILER_EVENT_SCOPE？GetElementShaderBindings那就是提一次Drawcall的那种费吧。。

- 地形绘制的drawcall是从哪提出去的：Scene的动态物体列表

- 草为什么提了那么多drawcall？HISM构建的问题

- 但怎么关联草和地形的渲染，难道不会出现地形没画草画了的情况吗？会的。。。

  <img src="Unreal Landscape.assets/image-20210815153931540-16290131776351.png" alt="image-20210815153931540" style="zoom: 25%;" />

- 渲染顺序？无法控制

- 地形lod后草按地形mipmap生成吗?没有，grassData里就保存了一个Heightmap数据，本来草和地形渲染也分开的

- LastLOD能设置吗?不是很行，值的填充在FLandscapeComponentSceneProxy中

- landscapeRendersystem和proxy什么关系呢,ALandscapeProxy和component的proxy什么关系呢

- 为什么每帧都要RecreateBuffers呢？因为要更新UniformBuffer参数

- LODIndexToMaterialIndex?UpdateMaterialInstances_Internal?

- Landscape VisibilityMask内部是什么，为什么可以无视材质实例的BlendMode Override？ 因为 ALandscape::PostEditChangeProperty中把MaterailInstance的bOverride_BlendMode给设成false了

- 为什么有个图总是传两遍？

- <img src="Unreal Landscape.assets/image-20210815183552713.png" alt="image-20210815183552713" style="zoom:25%;" />

  

![image-20210810202826489](E:\文件\SophisticatedLibrary\Unreal\Unreal Landscape.assets\image-20210810202826489.png)



### p6

- LOD切换：配置感觉挺好的了。但不可达块能不能开ForceLOD？比如ForceLOD2，因为现在的LOD切换配置最远也就差不多LOD2的样子 。尤其雨天场景，有些地形块利用效率很低。如果能因此少生成几级lod IndexBuffer就好了，但要改的地方太多了

- drawcall：雨天和新手村都是16个component的地形，drawcall基本在8个或以下，不知道能否再剔除掉一些

- 材质：仿佛不是瓶颈，并且因为占得面积大，并且是瓶颈也没办法

- 草的drawcall数高，簇分的太细了（但HISM的建树参数应该是全局的），顶点数也高。每个Component上有个hism，不可达的Component上最好一棵草都不要有。然后在Proxy的ShouldTickGrass上进行更严格的判断。

  grass.TickInterval能否改大

  由于我们是一关一关的游戏，，有些地形块是不是根本不需要加载GrassData。。

- lighting 部分能否只保留Static shadow？

- WeightMap是RGBA8格式，能换成有压缩的格式么？（好像不行？第一张那个Normal还放在里面，Normal有必要放在那里面么，不能放在顶点Normal里么，还导致有些地形块要两张WeightMap）

- 地形Component的渲染顺序 能否控制？现在是零散插在Mesh渲染中间的(不行吧)

- 地形不能烘焙成静态模型么？landscape做编辑用，烘焙的静态模型和Texture做渲染用，能省掉地形的tick，压缩解压缩，顶点着色器和地形 Texture等开销（切换lod可能会跳，physics Material、步迹系统可能受影响，可是远处也不需要那些啊。。）

### Reference

[Houdini Terrain & UE4](https://zhuanlan.zhihu.com/p/68062891)

[Houdini Terrian & UE4 （一）基础的地形工作流](https://zhuanlan.zhihu.com/p/68111850)

[Houdini Terrian & UE4 （二）mask & layer 制作流程优化](https://zhuanlan.zhihu.com/p/68628338)

[Unreal Engine地形系统辨析(三)](https://zhuanlan.zhihu.com/p/84531142)

[深度好文：UE4移动端地形理解 - 高度LOD](https://zwcloud.net/#blog/109)

[UE4 Mobile Landscape 总览及源码解析](https://zhuanlan.zhihu.com/p/144031549)

