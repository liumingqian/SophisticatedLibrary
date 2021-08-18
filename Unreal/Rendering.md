Shader Model 1.0（DirectX8.0）、Shader Model 2.0（DirectX9.0b）、Shader Model 3.0（DirectX9.0c）、Shader Model 4.0（DirectX10）、Shader Model 4.1（DirectX10.1）和Shader Model 5.0（DirectX11）。

### GBuffer

常用的是5纹理GBuffer：GBufferA.rgb = WorldNormal（Alpha通道用PerObjectGBufferData填充）；GBufferB.rgba=Metallic, Specular, Roughness, ShadingModelID; GBufferC.rgb是BaseColor和GBufferA0组合。GBufferD留给自定数据（deferred decal），GBufferE留给预处理的shadow因子。



## 剔除

四种剔除按开销由低到高进行，并构建一张逐模型的可见性列表。

Distance Culling：每个物体都有最大渲染距离属性，也可以用Cull Distance Volume影响整个Volume中的物体

Frustum Culling：视锥剪裁

precomputed Visibility：通过预构建生成，将场景划分为若干网格，摄像机进入网格后可以通过查询网格对应的表知道当前位置哪些物体可见哪些不可见，适用于中小型关卡，适用于低端硬件。

Occlusion Culling（动态遮挡/硬件遮挡查询）：

硬件遮挡剔除首先将遮挡体渲染到深度缓冲，然后创建一个遮挡查询（occlusion query），这一步用硬件深度测试渲染希望查询遮挡情况的模型，返回通过深度测试的像素数量。这一步渲染的是包围盒而不是原模型。然后将查询结果读回CPU，如果返回像素数为0则物体完全被遮挡，不提交模型给GPU渲染。CPU读回数据会延迟一两帧，因此物体可能会突然出现消失。

每个actor粒子通过boundingbox进行遮挡剔除

最后还会通过ShadowFrustomQueries提交针对光源的包围盒遮挡测试 ，如果光源被完全遮挡，则不进行光照和投影计算。PlanarReflectionQueries进行平面反射的遮挡剔除计算。

将若干模型合并成大模型虽然可以在遮挡剔除的时候降低CPU消耗，但很大的模型因为难以被遮挡剔除，因此会增加GPU消耗，且大物件Lightmap精度低。

大型开放世界，遮挡可能没太大作用，而计算遮挡的性能依然消耗了。

**HZB(Hierarchical z pass occlution）**

类似occlusion culling，使用Mip版本（Hi-Z Buffer？）的场景深度RT检查Actor边界，因此会剔除更少对象。仍然需要在Cpu上读取结果。

**软件遮挡查询/软光栅剔除** 

硬件遮挡查询在GPU进行可视性检查，软件遮挡查询在CPU上对场景进行光栅化来剔除Actor。它使用actor的指定LOD来遮挡后面的Actor。在移动平台上软件遮挡是单帧延迟，硬件遮挡查询则是双帧延迟。

使用方法是在project setting中启用软件遮挡，并且在Mesh编辑器中，将LOD for Occluder Mesh设置为0以上的值。可以只在中大型Mesh上做，小的网格体几乎不会成为遮挡物。

CPU不能像GPU一样快进行栅格化，因此这项技术的效率取决于需要栅格化的遮挡物的数量，其在屏幕上的大小，遮挡缓冲区的分辨率等因素

软光栅不支持hism

**Prepass/EarlyZ Pass**

pixel shader如果很重的话，渲染了而被剔除的pixel就很浪费，prepass是在光栅化之后，pixel shader之前预先按照深度进行渲染，可以渲染出一张mask，剔除掉每个物体被别的物体覆盖掉的像素，只渲染最终会出现在画面中的像素，因此是两倍draw call。earlyz是现代GPU硬件的标准流程，排序物体距离，按从前到后的顺序渲染物体，当不透明的图元从光栅化阶段开始逐像素处理后首先进行深度测试，不通过深度测试就不走片元着色器，Alpha Test或者Depth modify都会使early z失效，但对于不会抛弃像素的不透明物体来说earlyz的结果就是正确的。prepass可以配合earlyz使用，earlyz由于需要cpu端的物体排序，当场景非常复杂的时候频繁排序会导致性能下降，渲染prepass的话，earlyz就可以使用prepass的信息。

AlphaTest会让Early Z失效，因为Early Z是发生在片元着色器之前的，alphaTest会通过earlyZ，导致它后面的片元被抛弃，但该片元在片元着色器里通不过alphaTest。因此AlphaTest物体应用early z的方法是在z-prepass中渲染opaque和masked材质，只写深度，先渲染opaque，再渲染mask，渲染masked时候开启discard。第二个正式渲染的pass中将深度测试改为equal，关闭写深度的渲染opaque物体和masked物体，此时不开启discard，但因为用equal深度测试模式，因此masked的像素会正确通过。

[pre-z](https://zhuanlan.zhihu.com/p/81011375)

**HSR（Hidden Surface Removal）**

把着色延迟到所有图元都光栅化完成，得到fragment后，因此彻底解决了overdraw问题。earlyz虽然对物体进行了排序，但不能保证不发生像素遮挡，还是会有overdraw。

有和没有是tbdr（hsr）和tbr（early z)的区别，tbdr里的defer，defer的就是片元着色器。

hsr可以一直开到第一个透明物体或mask物体，然后后面就不开了。


## Virtual Texture
### RVT（运行时虚拟纹理）

**key**:2012年的技术，几乎没有用在手机上，因为手机带宽不够可能导致一些问题。使用CPU+内存解救GPU的技术。常用于大地形绘制。

比如有一个8k*8k的地形，不能一次性加载进内存，在内存中创建一个1K*1k的虚拟纹理，这张纹理上有若干Tile（Tile 有若干Lod尺寸），并配了一个look up table（LUT按Tile的最小lod尺寸索引，比如Tile最大lod尺寸为128*128，最小Lod尺寸为16*16，则LUT是一个64*64的表，每个元素保存某个地形块在VT中的位置（UVOffset））。

VT的工作流是，当视点不变时，只执行Pass B，采样3次左右。当视点改变时，需要将新的块加载进VT，执行Pass A后执行Pass B。当一个块不在视点中很久时，标记该块过时，后面的Tile可以blit到这个位置，覆盖它。

PassA：vertex shader后 ，一个地形块上可能有多个材质层，如石头上有青苔、泥土、雪等，将所有材质及其lightmap、reflection cube等贴图采样并烘焙成一个Tile并blit到VT上

PassB: vertex shader后 ，从LUT采样一次，从VT中采样Base Color和Normal（specular和roughness可能写进baseColor和Normal的alpha通道）渲染

传统工作流：vertex shader后 执行pixel shader，每帧都要执行对多个material layer的base color和normal采样，对于4个材质层的地形，是8次以上的采样。这个采样是固定的，为什么每帧都去采样呢，但如果将大地图全部烘焙保存在内存里，又承受不了内存开销，因此采用了类似虚拟内存的这样的技术，让视野中的块换入换出。

### SVT

RVT和SVT加载方式类似，都是以tile方式加载SVT作为asset是存在硬盘上的；RVT是当你在Run Time时会进行一个生成和渲染

**UV Dilation**

UV Dilatoin可以防止在较低的MipMap中出现EdgeBleeding

https://shaderbits.com/blog/uv-dilation

![img](Rendering.assets/clipboard.png)

## 抗锯齿

TAA可能导致抖动。因为是通过抖动画面并取平均进行模糊的。

MSAA只能在前向渲染使用。

Composite Texture功能可以通过增强法线转折处的Roughness，减小高光反射，从而减弱高光锯齿。该功能没有额外开销。



## Reflection

反射是间接光的高光，或者高光的GI。高光GI缺少阴影表现为反射漏光。反射对pbr效果非常重要。

### Reflection Capture

提供环境反射，只有间接高光且不包含漫反射。UE4在移动端的标准光照模型中SkyLight Cubemap和Reflection Capture Cubemap是互斥的（因为RGBM Encode的原因），且在选择时Relfection Capture的优先级高于Skylight，导致永远只可能Reflection Capture有效。移动端的Relfection Capture的范围也无效，物体在渲染时只取离它最近的那一个，也导致在场景制作过程中，需要区分室内室外，楼上楼下，多变的环境氛围时工作流几乎不可能实现，因为无法精确控制范围。

使用反射捕捉时，引擎会将反射捕捉的间接镜面反射与光照贴图的间接漫反射混合在一起。这有助于减少泄漏，因为反射立方体贴图仅在空间的一个点处捕获，但是光照贴图是在所有接收器表面上计算的，并且包含局部阴影。混合对于粗糙的表面效果很好，但是对于平滑的表面，Reflection Captures的反射与屏幕空间反射或平面反射等反射结果不能匹配，因此不能在光滑表面上进行光照贴图混合。

粗糙度为.3的曲面将得到完全的光照贴图混合，而粗糙度为.1或更低时，淡化为没有光照贴图混合。这样可以使Reflection Captures和SSR更好地匹配，并且很难发现过渡。开启reduce lightmap mixing on smooth surfaces选项可减少光滑平面的lightmap混合。

**Reflection capture actor**

是Static的，在building light的时候生成mipmap cubemap，只能反射静态的物体，运行时性能开销不大，但如果重叠很严重的话就会产生严重的overdraw，因为unreal会在多个reflection capture actor中间混合。另外半径越大开销可能会越大。半径小的优先级高于半径大的。

[UE4移动端预览模式下Reflection Capture调整后变黑问题](https://blog.csdn.net/StraightenupRyan/article/details/105241397)

[recapture优化](https://zhuanlan.zhihu.com/p/138078907)

### SSR

Basic SSR是Ray tracing算法，对于要反射的每一个像素，计算反射光线，然后在DepthBuffer中沿着反射光线方向进行trace，取交点处的颜色作为反射颜色。

SSR等需要开启高精度法线以减少拉伸。

手机上没有GBuffer的Normal，用不了SSR，新出的是SSPR

Unreal里面的反射效果有多种产生途径，比如设置roughness，比如把material的lighting mode设置为surface translucencyVolume。

planar reflection开销很大，效果很好。Screen Space Reflection比 planar好一点，但也贵，并且SSR只在后处理阶段在屏幕空间做，侧着看会效果不对，所以不适合用在镜子等上面，但是用在地板之类的地方效果不错。通常需要结合reflection actor和SSR等技术来产生反射效果。将多个立方体贴图进行混合的反射方法可能会导致反射中有重影，可以使用细节法线贴图、粗糙度等设置打破反射。



## 粒子系统

粒子效果单独渲染到一个Render Target上。

GPU粒子可以支持Depth Buffer Collision，但是只有translucent支持，[mobile不支持gpu sprite碰撞](https://docs.unrealengine.com/en-US/RenderingAndGraphics/ParticleSystems/Reference/TypeData/GPUSprites/index.html)

## 后处理

首先以HDR渲染场景，然后根据颜色直方图和EyeAdaption、SceneColorTInt等设置，tonemapper将HDR颜色映射为LDR颜色，然后color grading是把一种LDR颜色映射为另一种LDR颜色

TAA（使用History Buffer和Velocity Buffer）、运动模糊、自动曝光（计算场景亮度直方图，计算每个亮度区间内有多少像素，相应调整曝光

Bloom：为超过阈值的颜色区域添加高斯模糊，产生泛光

Tonemapping：这一步骤可以为不支持HDR的设备进行色调映射。UE支持mobileHDR渲染，但不支持HDR Color Buffer，无法进行Mobile HDR输出。

SceneColor.w保存msaa后的depth

一般postProcessMaterial都是在tonemapping之后，因为之前是HDR，更加浪费带宽。HDR是一个像素32位（8bytes的），LDR是16位（4bytes）

ppv的混合经测试，可以混合后处理参数，但是ppv material只能在两个父材质相同的材质实例中间混合，嵌套ppv的情况，如果外面使用了Material，里面不使用material也无法取消Material的效果。
