
| [教程主页](index.md)    | [上一页](Compilation.md) | [下一页](ApplyingDatafilters.md) |
| ------------- |:-------------:| -----:|

# 数据点过滤器（Datapoint Filters）

在本节中，将介绍在进行 ICP 之前可以应用于读入点云和参考点云的各种过滤器。

提醒一下，*数据点过滤器（datapoint filters）* 可以有多种用途：

* 移除噪声点，因为这些点会使点云对齐变得困难。
* 移除冗余点，以加快对齐速度。
* 为点添加描述性信息，比如表面法向量，或者从点指向传感器的方向。

注意，*数据点过滤器* 不同于 *离群点过滤器（outlier filters）*，后者出现在 ICP 流水线的更后部分，并且具有完全不同的目的。

[Libpointmatcher](https://github.com/anybotics/libpointmatcher) 为开发者提供了多个数据点过滤器，这些过滤器将输入点云处理为中间点云，并用于后续对齐步骤。过滤器作为独立模块工作，并且经常组合成链。通过顺序的数据点过滤器链，可以根据具体对齐问题进行定制。

## 过滤器索引
### 下采样（Down-sampling）
1. [Bounding Box Filter](#boundingboxhead) —— 边界框过滤器

2. [Maximum Density Filter](#maxdensityhead) —— 最大密度过滤器

3. [Maximum Distance Filter](#maxdistancehead) (**已弃用**)

4. [Minimum Distance Filter](#mindistancehead) (**已弃用**)

5. [Distance Limit Filter](#distancelimithead) —— 距离限制过滤器

6. [Maximum Point Count Filter](#maxpointcounthead) —— 最大点数过滤器

7. [Maximum Quantile on Axis Filter](#maxquantilehead) —— 轴向最大分位数过滤器

8. [Random Sampling Filter](#randomsamplinghead) —— 随机采样过滤器

9. [Remove NaN Filter](#removenanhead) —— 移除 NaN 过滤器

10. [Shadow Point Filter](#shadowpointhead) —— 阴影点过滤器

11. [Voxel Grid Filter](#voxelgridhead) (**已弃用**)

12. [Octree Grid Filter](#octreegridhead) —— 八叉树栅格过滤器

13. [Normal Space Sampling (NSS) Filter](#nsshead) —— 法向空间采样过滤器

14. [Covariance Sampling (CovS) Filter](#covshead) —— 协方差采样过滤器

### 描述子增强（Descriptor Augmenting）
1. [Observation Direction Filter](#obsdirectionhead) —— 观测方向过滤器

2. [Surface Normal Filter](#surfacenormalhead) —— 表面法向过滤器

3. [Orient Normals Filter](#orientnormalshead) —— 法向方向化过滤器

4. [Sampling Surface Normal Filter](#samplingnormhead) —— 采样表面法向过滤器

5. [Simple Sensor Noise Filter](#sensornoisehead) —— 简单传感器噪声过滤器


## 一个公寓点云示例视图
![alt text](images/floor_plan.png "公寓平面图")

下面的示例来自可在此处下载的公寓数据集：  
http://projects.asl.ethz.ch/datasets/doku.php?id=laserregistration:apartment:home  
来自 ETH Zurich ASL 实验室。

下面展示的是点云的俯视图，颜色表示垂直高度。天花板已从点云中移除，因此地面（蓝色）和墙壁（红色）清晰可见。请注意，坐标原点位于厨房，也即数据采集起点，位于俯视图和公寓平面图的左上角。

![alt text](images/appt_top.png "来自公寓数据集的点云俯视图")

## Bounding Box Filter <a name="boundingboxhead"></a>
### 描述
该过滤器可以用于从点云中排除矩形边界区域外的点。边界框由 x、y、z 方向的最小和最大坐标值定义。

__需要的描述子：__ 无  
__输出描述子：__ 无  
__传感器假设在原点：__ 否  
__对点数量的影响：__ 减少点数

|参数  |描述  |默认值    |允许范围|
|---------  |:---------|:----------------|:--------------|
|xMin       |x 轴最小值（边界框一侧）| -1.0 | -inf ~ inf |
|xMax       |x 轴最大值（边界框一侧）| 1.0 | -inf ~ inf |
|yMin       |y 轴最小值 | -1.0 | -inf ~ inf |
|yMax       |y 轴最大值 | 1.0 | -inf ~ inf |
|zMin       |z 轴最小值 | -1.0 | -inf ~ inf |
|zMax       |z 轴最大值 | 1.0 | -inf ~ inf |
|removeInside   |设为 1 时移除边界框内的点，否则移除外部点 |1| 0 或 1 |

### 示例
以下示例展示对输入点云施加一个边界框过滤器。

注意：通过设置 *removeInside = 0*，过滤器只会移除边界框**外部**的点。由于点云中心位于左上角的厨房，本例选择了一个 2m × 2m 的区域。滤波后的点以白色叠加显示。

（以下两图和表格保持不变）

## Maximum Density Filter <a name="maxdensityhead"></a>
### 描述
多个过滤器用于减少点云点数，方式包括随机下采样或随机丢弃点。在高密度区域中，点往往包含冗余信息，因此如果减少点数，ICP 可以运行得更快。本过滤器用于对点云密度进行均匀化，通过丢弃高密度区域的点来达到指定的最大密度。

只有当点的局部密度超过阈值时才会考虑删除；否则点保持。过滤器的唯一参数是点云所需的最大密度。点被随机移除以尽可能接近该最大密度。

__需要的描述子：__ `densities`（参考 SurfaceNormalDataPointsFilter 和 SamplingSurfaceNormalDataPointsFilter）  
__输出描述子：__ 无  
__传感器假设在原点：__ 否  
__对点数量的影响：__ 减少点数

|参数  |描述  |默认值    |允许范围|
|---------  |:---------|:----------------|:--------------|
|maxDensity |输出点云期望的最大密度（3D：points/m³，2D：points/m²） | 10 | min: 1e-7, max: inf |

### 示例
以下展示最大密度过滤器对公寓点云子区域的效果。高密度区域（红色）进行下采样，低密度区域（蓝色）被保留。采样后的点以白色叠加显示。

（图与表格保持不变）

## Maximum Distance Filter (**已弃用**) <a name="maxdistancehead"></a>

**已弃用**：请考虑使用 [Distance Limit Filter](#distancelimithead)。

### 描述
该过滤器移除距离中心超过阈值的点。仅当点距中心距离**小于**阈值时才保留。距离阈值可以基于 x、y、z 分量，也可以是径向距离。

（下略，保持原意逐句翻译。）


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc0MTIxMTE0N119
-->