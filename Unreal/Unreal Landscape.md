### Landscape Basic

一个Component有1或4个section，每个Section有2^n-1个quad(顶点数是2^n)，Quad是最小单元格，默认一个1米，除非改了Scale。地形以Section为单位进行LOD，如果Sectionsde ScreenSize差异不大，会自动合并Drawcall。Component是渲染单位，每个component有一个高度图纹理，因此顶点数必须是2的幂。相邻Component边缘的共享顶点会复制一份，因此考虑每个Component中的顶点数量是有意义的。较小的component大小可实现更快的LOD过渡，也可实现更多地形的遮挡，但是对于一个固定大小的地形，component越小，所需要的component越多，使用更少的component通常可以获得更好的性能。

Unreal的地形实现基于geometry mipmapping，每个地形块有自己的Weightmap和Heightmap，地形块间耦合度很小，只是需要获取4个邻居的lod值，传给vertex shader用于做缝合。Viwo的做法比较像Unity。

LandscapeLayerCoords 节点生成UV坐标，可用于将材质网络映射到地形地貌。

LandscapeVisibilityMask 节点移除地形的可见部分，以便在地形中创建洞进行如洞穴创建等操作。

Edit layers需要在Details面板打开Enable edit layers。

Unreal使用 -256 到 255.992 (cm)之间的值计算高度图的高度，并以 16 位精度存储，然后将该值乘以地形的Scale.Z

#### 地形混合

- 基于权重：每个layer的颜色直接乘以该层权重值，然后再依次把所有算好的颜色值叠加一起
- 基于Height：在每层原始权重的基础上再叠加上一个来自与颜色图匹配的高度图的灰度值，后面会做个重新的归一化，这样就能保证输出的颜色亮度不会超过颜色纹理中的最高亮度。由于Weightmap分辨率只能与顶点数保持一致，这种混合方法比较能提升混合的细节。
- 基于Alpha：依照顺序逐层进行线性插值

编辑器中在对weightmap修改后，系统会做一个简单的判断，如果发现该层的权重值都变为零时，就把当前层移除掉。接着它会更新这个component的材质实例，根据当前的layer的使用情况生成对应的shader变体，假如有新的一层添加进来也会做同样的处理。

[grasstype](https://www.youtube.com/watch?v=wE2_CKG6vRQ)

### 地形草

每个landscape component都分配了一个专属的hism。如果某个landscape component不能生成草的实例，那么就不会创建hism。grass的hism是在游戏运行时构建的，并不会序列化到资产文件里。所以每帧系统都会轮询世界中所有的landscapeproxy，判断landscapeproxy中的每一个landscape component是否在预设的渲染范围之内。如果这个在范围之内的component没有创建过hism，就会开启一个异步任务用来构建hism的cluster tree，并且形成对应的instance buffer。每个instance的位置会重新进行随机，但是为了保证随机的一致性，构建时使用的随机种子都是和landscape component一一对应的，所以同个位置上的hism里的instance每次重新生成，都会出现在一模一样的地方。grass data里并没有记录着instance的信息，只是grass type相关的weightmap和heightmap，weightMap帮助确定instance的位置，heightmap帮助instance对齐地形表面的高度和斜率。

[Unreal Engine的地形草管理系统的缺陷分析报告](https://zhuanlan.zhihu.com/p/149771853)

ES3.1以上会通过CreateGrassIndexBuffer创建一个草的IndexBuffer。

### 编辑

地形编辑器主要逻辑在LandscapeEdit.cpp。编辑时每个Component的数据存储在 ULandscapeComponent::PlatformData中。PlatformData中存储的顶点数据格式是FLandscapeMobileVertex，每个顶点对应一个FLandscapeMobileVertex：

```cpp
struct FLandscapeMobileVertex
{
	uint8 Position[4]; //x,y，所在Section在Component内的坐标，挖洞的信息
	uint8 LODHeights[LANDSCAPE_MAX_ES_LOD_COMP*4];//0和1存储所有LOD高度中的最大最小值（抛弃掉了uint16的低八位），各个LOD的高度归一化到最大最小高度范围内，存储到LODHeights[2+Mip]中。
};
```

ULandscapeComponent::GeneratePlatformVertexData为Mobile生成了PlatformData，用HeightMapTexture的Mip获取各个LOD的高度数据，填充到FLandscapeMobileVertex数组里。

### RHI数据创建和索引

#### SharedBuffer

运行时渲染数据由FLandscapeComponentSceneProxy::CreateRenderThreadResources()创建，填充在FLandscapeSharedBuffers里。FLandscapeSharedBuffers中包含了Indexbuffer(每个LOD一个IndexBuffer),VertexBuffer等数据和LOD参数（以UniformBuffer的形式）。FLandscapeSharedBuffers保存在一个全局变量SharedBuffersMap里，由SharedBuffersKey索引。

SharedBuffersKey只由SubsectionSizeQuads、NumSubsections唯一确定，因为属于一个Landscape的Component的SubsectionSizeQuads、NumSubsections是一模一样的，所以顶点分布（或者说顺序）也是一模一样的，所以没必要重复为每个Component生成不同的IndexBuffer，只要SubsectionSizeQuads、NumSubsections两个参数相同，就可以复用之前生成好的Index Buffer，因此所有Component的Proxy公用一个SharedBuffersKey。VertexBuffer是per proxy有自己的。

顶点排列根据有1个或4个sections，有两种情况：

<img src="Unreal Landscape.assets/vertex_order_section1x1.png" alt="Component内顶点的排列方式1x1" style="zoom:50%;" />

<img src="Unreal Landscape.assets/vertex_order_section2x2.png" alt="Component内顶点的排列方式2x2" style="zoom:50%;" />

##### index buffer

FLandscapeSharedBuffers::CreateIndexBuffers生成了IndexBuffer。ES3.1下根据Fortyth算法进行了[Post Transform Cache](https://www.khronos.org/opengl/wiki/Post_Transform_Cache)的优化，并且在LOD间共用了顶点以节省内存。

如果顶点数超过65535，则IndexBuffer的indexType是uint32，否则是uint16。

实际生成RHI资源发生在`FRawStaticIndexBuffer16or32<INDEX_TYPE>::InitRHI()`中，里面

- 创建了RHI Index buffer: `RHICreateIndexBuffer` -> `glBufferData`，这里的实现就是创建一个buffer对象来保存index
- 创建了SRV(Shader Resource View): `RHICreateShaderResourceView` -> `glGenTextures`、`glTexBuffer`/`glTexBufferRange`。

##### vertex buffer

FLandscapeComponentSceneProxy::CreateRenderThreadResources中创建了FLandscapeVertexFactoryMobile，其数据用之前反序列化PlatformData得到的MobileRenderData中的VertexData填充。FLandscapeVertexFactoryMobile中最主要的数据是FDataType，FDataType在PC上的定义是

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

创建的LandscapeVertexFactory会保存在Proxy内，实际渲染时使用。、

##### uniform buffer

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

这些Uniform参数在FLandscapeRenderSystem::RecreateBuffers中赋值。

### 渲染

渲染循环：

```cpp
FMobileSceneRenderer::Render
FMobileSceneRenderer::InitViews
FPersistentUniformBuffers::UpdateViewUniformBuffer
FLandscapePersistentViewUniformBufferExtension::BeginRenderView
FLandscapeRenderSystem::BeginRenderView
FLandscapeRenderSystem::RecreateBuffers
```

Vertex shader伪代码：

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

### LOD

lod数是SubsectionSizeQuads以2为底的对数，即如果15*15个Quads构成一个section，NumLod为log(2,16)=4。

LOD参数在ALandscape的Detail窗口中指定：

- LOD0ScreenSize:
- LOD0:
- Other Lods:减少远处地形的顶点数（会影响到FractionalLOD)，不是线性的

移动平台应该是支持PLATFORM_SUPPORTS_LANDSCAPE_VISUAL_MESH_LOD_STREAMING的

FLandscapeRenderSystem中的SectionLODSettings为每个Section保存了一个LODSettingsComponent。FLandscapeRenderSystem::ComputeSectionPerViewParameters中根据SectionLODSettings、ScreenSize和LODScale计算FractionalLOD，最后将所有Section的LOD值作为一个数组存在CachedSectionLODValues中。最终保存到上面的Uniform Buffer中FLandscapeSectionLODUniformParameters::SectionLOD。在shader中需要获取相邻SectionLOD的数据，知道Section的坐标，也就能通过偏移拿到相邻Section的LOD了。

### 剔除

CreateOccluderIndexBuffer



### Reference

[Houdini Terrain & UE4](https://zhuanlan.zhihu.com/p/68062891)

[Houdini Terrian & UE4 （一）基础的地形工作流](https://zhuanlan.zhihu.com/p/68111850)

[Houdini Terrian & UE4 （二）mask & layer 制作流程优化](https://zhuanlan.zhihu.com/p/68628338)



[UE4移动端地形理解 - 高度LOD](https://zwcloud.net/#blog/109)

地形哪里费：TRACE_CPUPROFILER_EVENT_SCOPE

UE做了哪些优化

剔除和LOD，绘制顺序如何影响地形效率

landscapeRender和Mobile区别：

TexCoordOffsetParameter

![image-20210810202826489](E:\文件\SophisticatedLibrary\Unreal\Unreal Landscape.assets\image-20210810202826489.png)

[Unreal Engine地形系统辨析(三)](https://zhuanlan.zhihu.com/p/84531142)