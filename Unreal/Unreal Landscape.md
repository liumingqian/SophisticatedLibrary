### Landscape Basic

一个Component有1或4个section，每个Section有2^n-1个quad(顶点数是2^n)，Quad是最小单元格，默认一个1米，除非改了Scale。地形以Section为单位进行LOD，如果Sectionsde ScreenSize差异不大，会自动合并Drawcall。Component是渲染单位，每个component有一个高度图纹理，因此顶点数必须是2的幂。相邻Component边缘的共享顶点会复制一份，因此考虑每个Component中的顶点数量是有意义的。较小的component大小可实现更快的LOD过渡，也可实现更多地形的遮挡，但是对于一个固定大小的地形，component越小，所需要的component越多，使用更少的component通常可以获得更好的性能。

Unreal的地形实现基于geometry mipmapping，每个地形块有自己的Weightmap和Heightmap，地形块间耦合度很小，只是需要获取4个邻居的lod值，传给vertex shader用于做缝合。Viwo的做法比较像Unity。

LandscapeLayerCoords 节点生成UV坐标，可用于将材质网络映射到地形地貌。

LandscapeVisibilityMask 节点移除地形的可见部分，以便在地形中创建洞进行如洞穴创建等操作。

Edit layers需要在Details面板打开Enable edit layers。

#### 地形混合

- 基于权重：每个layer的颜色直接乘以该层权重值，然后再依次把所有算好的颜色值叠加一起
- 基于Height：在每层原始权重的基础上再叠加上一个来自与颜色图匹配的高度图的灰度值，后面会做个重新的归一化，这样就能保证输出的颜色亮度不会超过颜色纹理中的最高亮度。由于Weightmap分辨率只能与顶点数保持一致，这种混合方法比较能提升混合的细节。
- 基于Alpha：依照顺序逐层进行线性插值

编辑器中在对weightmap修改后，系统会做一个简单的判断，如果发现该层的权重值都变为零时，就把当前层移除掉。接着它会更新这个component的材质实例，根据当前的layer的使用情况生成对应的shader变体，假如有新的一层添加进来也会做同样的处理。

grasstype：https://www.youtube.com/watch?v=wE2_CKG6vRQ

### 地形草

每个landscape component都分配了一个专属的hism。如果某个landscape component不能生成草的实例，那么就不会创建hism。grass的hism是在游戏运行时构建的，并不会序列化到资产文件里。所以每帧系统都会轮询世界中所有的landscapeproxy，判断landscapeproxy中的每一个landscape component是否在预设的渲染范围之内。如果这个在范围之内的component没有创建过hism，就会开启一个异步任务用来构建hism的cluster tree，并且形成对应的instance buffer。每个instance的位置会重新进行随机，但是为了保证随机的一致性，构建时使用的随机种子都是和landscape component一一对应的，所以同个位置上的hism里的instance每次重新生成，都会出现在一模一样的地方。grass data里并没有记录着instance的信息，只是grass type相关的weightmap和heightmap，weightMap帮助确定instance的位置，heightmap帮助instance对齐地形表面的高度和斜率

[Unreal Engine的地形草管理系统的缺陷分析报告](https://zhuanlan.zhihu.com/p/149771853)

地形为什么费

UE做了哪些优化

剔除和LOD如何影响地形效率

landscapeRender和Mobile区别：

TexCoordOffsetParameter

![image-20210810202826489](E:\文件\SophisticatedLibrary\Unreal\Unreal Landscape.assets\image-20210810202826489.png)

[Unreal Engine地形系统辨析(三)](https://zhuanlan.zhihu.com/p/84531142)



### Reference

[Houdini Terrain & UE4](https://zhuanlan.zhihu.com/p/68062891)

[Houdini Terrian & UE4 （一）基础的地形工作流](https://zhuanlan.zhihu.com/p/68111850)

[Houdini Terrian & UE4 （二）mask & layer 制作流程优化](https://zhuanlan.zhihu.com/p/68628338)



[UE4移动端地形理解 - 高度LOD](https://zwcloud.net/#blog/109)