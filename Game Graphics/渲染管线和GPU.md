### TBDR

##### key



##### 步骤

1. 生成图元列表，确定每个Tile上有哪些primitive，列表保存在CPU上
2. 逐个tile执行光栅化及后续处理，完成后将Frame Buffer从Tile Buffer写回System Memory

##### 优劣

- 可以执行消除Overdraw的优化，PowerVR用了HSR技术，Mali用了Forward Pixel Killing技术。-

- 在Cache读写毒素比全局内存快很多。降低带宽，减少功耗。（移动平台功耗比pc差几十倍，带宽差十倍左右）
- 几何数据多的话可能在输出几何数据写入内存，fragmentshader读取的阶段产生瓶颈
- 在多个tile上的三角形要绘制数次

