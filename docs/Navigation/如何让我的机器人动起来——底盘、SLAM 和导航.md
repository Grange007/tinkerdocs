﻿---
last_update:
  date: 4/29/2024
  author: Grange
tag: Tinker青训-2024春
---

# 如何让我的机器人动起来——SLAM、导航与底盘

## Overview

解决的核心问题：如何让小车自动移动到指定位置

*一个简单的问题由于实际环境的复杂导致存在大量的误差*

我的周围可能存在很多的障碍物，路径存在限制，我要怎么知道我周边环境的地图信息？

前往目标点的路径不一定是直线，我要怎么根据现有地图找到最优路径？

我知道小车的整体目标速度，要怎么控制电机才能让整个小车按照我的目标速度行驶？



**SLAM**：得到环境的整个地图

**导航**：根据已有的环境地图，输入要到达的目标点，根据算法输出在小车行驶过程中每个时刻的底盘速度

**底盘**：将导航计算出的底盘速度通过运动学结算转化为每个电机的速度，再控制每个电机按照目标速度行驶



*由于 SLAM 和导航都是非常庞杂的工程框架，本次培训将**不涉及**对具体代码的讲解，而以**整体框架**的介绍为主*

*相信在听懂整体框架后，后续上手也不会太难* :smile:



## SLAM

### 什么是 SLAM

SLAM 是 **Simultaneous Localization And Mapping** 的缩写，中文译作 ”**同时定位与地图构建**“ 。

它是指搭载特定**传感器**的主体，在**没有环境先验信息**的情况下，于**运动过程**中建立**环境**的模型，同时估计自己的**运动**。

在实际应用中，SLAM 的过程一般会先于导航完成，通过遥控 or 自动行驶的方式让机器人先完成对环境的建图。

[利用Cartographer SLAM算法跑德意志博物馆bag效果](https://www.bilibili.com/video/BV1tS4y1S7qE/?spm_id_from=333.1350.jump_directly&vd_source=66fdb6136aee8c29f464c123e245cc80)

需要实现的功能：

1. 我在什么地方？——定位
2. 周围环境是什么样？——建图

只有时刻明白自己在地图中的相对位置，机器人才能一边移动一边还原出地图的全貌。



### SLAM 如何实现建图定位问题

![经典视觉SLAM结构](https://fishros.com/d2lros2/foxy/chapt10/10.3SLAM%E6%8A%80%E6%9C%AF%E6%A6%82%E8%BF%B0/imgs/image-20220421152216184.png)

**经典 SLAM 流程：**

1. **传感器信息读取**：通过各类传感器实现对环境的感知，比如通过激光雷达获取环境的深度信息。同时可以通过传感器融合来提高位置估计的精度，比如融合轮式里程计、IMU、雷达、深度相机数据等。
2. **前端里程计**（视觉/激光）：根据相机获得的图像数据或激光雷达获得的点云数据，通过前后数据之间对比得出机器人位置的变化。
3. **后端优化**：对不同时刻下前端里程计和回环检测的信息进行整合和优化，得到全局一致的轨迹和地图。
4. **回环检测**：判断机器人是否到达过先前经过的位置，从而解决位置估计误差问题。
5. **建图**：将 SLAM 得到的环境信息作为地图存储下来，机器人之后在环境中的运动就可以通过已知的地图来导航了。



#### 传感器

**激光雷达**

感兴趣的同学可以看一下我写的激光雷达概览：[LiDAR 初探 | Grange's blog (grange007.github.io)](https://grange007.github.io/2024/03/04/OverviewOfLidar/)

基本原理：

+ 向被测物体射出一束激光，通过对激光方向、传播时间的计算，得到物体目前的位置
+ 通过旋转的方式可向四周射出激光，从而获得周围环境的位置数据
+ 最终得到离散的点云数据（每个点代表这个位置有个东西），点云的密度取决于激光雷达的帧率和点频

![](https://blog.generationrobots.com/wp-content/uploads/2019/08/what-is-a-lidar-part-1-9.jpg)
![](如何让我的机器人动起来——SLAM、导航与底盘\pointclouds.png)



**摄像头**

通过相机拍摄的图像得到环境中物体相对机器人的位置（物体在图像中的深度）

+ 单目相机：比较难以准确确定物体的深度

+ 双目相机：由左右两个单目相机组成，可通过比较左右眼图像中同一像素的位置差异判断物体的远近

![](https://robot.czxy.com/docs/camera/chapter03/assets/image-20200403161101821.png)

+ 深度相机：相机内置红外结构光或激光传感器，可直接测出与物体的距离



#### 前端里程计

![](如何让我的机器人动起来——SLAM、导航与底盘\cameramove.png)

+ 通过相邻帧间的图像估计相机运动，并恢复场景的空间结构（图像特征匹配+几何位姿计算）

+ 具有短时记忆，只能计算相邻时刻的运动

+ 由于误差，可能存在累计偏移的问题，称为**漂移**（e.g. 每次前端里程计估计出来的物体方向都比实际向左偏了1°）

![](如何让我的机器人动起来——SLAM、导航与底盘\slamdrift.png)

#### 后端优化

收到的数据中存在**噪声**，要考虑如何从带有噪声的数据中估计整个系统的状态——滤波和非线性优化算法

*需要较多的数学基础，前面的道路留给之后再探索吧~*



#### 回环检测

主要解决位置估计随时间漂移的问题

假设机器人经过一段时间的运动后回到了原点，但由于漂移，它计算出的位置估计值却没有回到原点，那么就应该把它计算的位置给拉回原点

主要通过视觉与机器学习方法完成：判断当前图像与历史图像的相似性，从而辨认出它们来自同一个地方

#### 建图

![](如何让我的机器人动起来——SLAM、导航与底盘\map.png)

**3D & 2D 地图**

+ 存储地图的三维/二维位置数据

**尺度地图**

+ 把真实世界按比例缩小，直接表示在地图上

​	**栅格地图表示方法**：

+ 由许多小方块划分地图，用每一个方块的数值来表示地图信息

  传感器存在噪声，机器人前方的某个位置到底有没有物体（障碍物）是不确定的。

  采用概率**（占据率）**来解决这一问题

  + 认为确实有物体的栅格的占据率为100%

  + 确定没有物体的栅格占据率为0%，

  + 不确定的栅格就用（确认占据概率/确认非占据概率）值表示占据率。

    ![image-20220506134542099](https://fishros.com/d2lros2/humble/chapt10/basic/2.%E6%A0%85%E6%A0%BC%E5%9C%B0%E5%9B%BE%E4%BB%8B%E7%BB%8D/imgs/image-20220506134542099.png)

**拓扑地图**

+ 只由节点和边组成

+ 强调地图元素之间的连通关系

  ![image-20240426204233622](如何让我的机器人动起来——SLAM、导航与底盘\topology.png)

  

![经典视觉SLAM结构](https://fishros.com/d2lros2/foxy/chapt10/10.3SLAM%E6%8A%80%E6%9C%AF%E6%A6%82%E8%BF%B0/imgs/image-20220421152216184.png)



## 导航

在获得地图之后，导航要实现的就是从地图上找到距离目标点的最优路径，并将这一路径发送给底盘。如果说 SLAM 解决的是“我在哪”，导航就是要解决“我要怎么走”。

导航的具体实现方式有很多，Tinker 目前采用 [Nav2](https://navigation.ros.org/) 作为导航框架，以下导航内容也将基于 Nav2 进行讲解。

![navigation_fishbot2](https://fishros.com/d2lros2/humble/chapt11/get_started/3.%E4%BD%BF%E7%94%A8FishBot%E8%BF%9B%E8%A1%8C%E8%87%AA%E4%B8%BB%E5%AF%BC%E8%88%AA/imgs/navigation_fishbot2.gif)

![fishbot_multi_point](https://fishros.com/d2lros2/humble/chapt11/get_started/3.%E4%BD%BF%E7%94%A8FishBot%E8%BF%9B%E8%A1%8C%E8%87%AA%E4%B8%BB%E5%AF%BC%E8%88%AA/imgs/fishbot_multi_point.gif)



### Nav2 的整体架构

![Navigation2框架图](https://fishros.com/d2lros2/humble/chapt11/get_started/1.Nav2%E5%AF%BC%E8%88%AA%E6%A1%86%E6%9E%B6%E4%BB%8B%E7%BB%8D/imgs/architectural_diagram-16525447663514.png)

### 传入数据

`map`：先前通过 SLAM 完成的建图

`Waypoint Foller`：发送导航目标点

`TF Transforms`：将传感器的坐标转化到世界坐标

`Behavior Tree`：对导航行为架构的描述

`Sensor Data`：导航过程中同时获取的传感器数据（虽然已有了先前的 SLAM 地图，但动态环境下可能会临时出现障碍物，需要实时地更新/重做 SLAM ）



### 四个服务

对上面的架构图进一步的进行解释，可以简单分为一大三小四个服务。

#### BT Navigator Server 导航行为树服务器

可以理解为组织机器人行为顺序的架构，（详情参考[这篇文章](https://arxiv.org/pdf/1709.00084)），通过这个大的服务来进行下面三个小服务组织和调用。



#### Planner Server 规划服务器

通过路径搜索函数，计算完成一些目标函数的路径。（可以近似理解为最短路问题，如 Dijkstra、A* ）



**全局代价地图 （Global Costmap）**：

全局代价地图主要用于全局的路径规划器。从上面结构图中其在可以看到在 Planner Server 中。

通常包含的图层有：

- Static Map Layer：静态地图层，通常都是SLAM建立完成的静态地图。

- Obstacle Map Layer：障碍地图层，用于动态的记录传感器感知到的障碍物信息。

- Inflation Layer：膨胀层，在以上两层地图上进行膨胀（向外扩张），以避免机器人的外壳会撞上障碍物。

![](https://assets.leetcode.com/uploads/2021/02/18/example2_1.png)

![](https://s3.amazonaws.com/media-p.slid.es/uploads/1476635/images/10738671/global_planner.png)


#### Controller Server 控制服务器

在ROS 1中也被称为局部规划器，是我们跟随全局计算路径或完成局部任务的方法。（结合机器人具体结构细节控制机器人按照 Planner Server 给出的路径来行驶）

![img](https://s3.amazonaws.com/media-p.slid.es/uploads/1476635/images/10738702/local_plan.png)

![img](https://s3.amazonaws.com/media-p.slid.es/uploads/1476635/images/10738701/restrictions.png)

![img](https://www.guyuehome.com//Uploads/Editor/202105/20210512_85734.png)

**局部代价地图（Local Costmap）**

局部代价地图主要用于局部的路径规划器。从上面结构图中其在可以看到在 Controller Server 中。

通常包含的图层有：

- Obstacle Map Layer：障碍地图层，用于动态的记录传感器感知到的障碍物信息。
- Inflation Layer：膨胀层，在障碍地图层上进行膨胀（向外扩张），以避免机器人的外壳会撞上障碍物。



#### Recovery Server 恢复服务器

恢复器是容错系统的支柱。恢复器的目标是处理系统的未知状况或故障状况并自主处理这些状况。说白了就是机器人可能出现意外的时候想办法让其正常，比如走着走着掉坑如何爬出来。



### 传出数据

`cmd_vel`：发送给底盘的目标速度数据

```json
geometry_msgs/Twist 
linear:
  x: 0.5
  y: 0.0
  z: 0.0
angular:
  x: 0.0
  y: 0.0
  z: 0.0
```

通过这一数据，我们就能告诉底盘目标的速度了！





## 底盘

到这里，我们还需要最后一步，就是让底盘按照我们的目标速度行驶。

要做到这一点，我们需要完成两部分功能：

1. 完成从底盘到电机的运动学结算
2. 控制电机按照目标速度转动



### 底盘运动学结算

**正运动学结算**：由电机速度计算得到底盘整体速度

**逆运动学结算**：由底盘整体速度反推得到电机速度

底盘运动学结算需要完成的功能就是将底盘整体的速度转化为每个电机的速度，我们是通过电机的速度来控制底盘的

在此我们以两轮差速模型为例作为讲解

![img](https://www.guyuehome.com//Uploads/Editor/202105/20210512_85734.png)

运动特性为两轮差速驱动，其底部两个同构驱动轮的转动为其提供动力

![img](如何让我的机器人动起来——SLAM、导航与底盘\diffmodel.png)

机器人的运动简化模型如图 所示，$X$ 轴正方向为前进、$Y$ 轴正方向为左平移、$Z$ 轴正方向为逆时针。机器人两个轮子之间的间距为 $D$，机器人 $X$ 轴和 $Z$ 轴的速度分别为：$V_x$ 和 $V_z$，机器人左轮和右轮的线速度分别为：$V_l$ 和 $V_r$，角速度分别为：$\omega_l$ 和 $\omega_r$，机器人轮子的半径为 $R$ 。

假设机器人往一个左前的方向行进了一段距离，设机器人的右轮比左轮多走的距离近似为 $K$,以机器人的轮子上的点作为参考点做延长参考线，可得：$\theta_1=\theta_2$。由于这个 $\Delta t$ 很小，因此角度的变化量 $\theta_1$ 也很小，因此有近似公式：
$$
\theta_2\approx sin(\theta_2)=KD
$$
由此可以计算出下面的式子：
$$
K=(V_r-V_l)*\Delta t
$$

$$
\omega = \theta / \Delta t
$$
由上面的公式和式子可以求解出运动学正解的结果：

机器人 $X$ 轴方向速度
$$
V_x=(V_l+V_r)/2
$$
机器人 $Z$ 轴方向速度:
$$
V_z=(V_r-V_l)/D
$$
由正解直接反推得出运动学逆解的结果：

机器人左轮的速度:
$$
V_l=V_x-(V_z*D)/2 \\
\omega_l=V_l/R
$$
机器人右轮的速度:
$$
V_r=V_x+(V_z*D)/2 \\
\omega_r = V_r/R
$$
理论上，如果我们能控制每个电机按照这里解算出的速度转动，就能让底盘按照我们预期的速度行驶了！



### PID 控制器

由于篇幅问题，PID 算法的具体原理不会在这里过多展开，感兴趣的同学可以参考[这篇文章](https://blog.csdn.net/qq_41415238/article/details/122079899)。

我们可以把电机看作一个系统，向其输入一个电流，电机会输出一个转速，也就是我们需要的角速度 $\omega$



**开环控制**

开环控制指输入不依赖于输出的控制方式，也是最直观的控制方式。假设我们已经知道了电机转速与电流的对应关系，我们只需要输入一个对应电流，即可获得我们的目标速度。

然而，实际情况可能会比这复杂很多，开环控制往往难以达到预期的精度。



**闭环控制**

闭环控制指输出会反馈给输入端从而影响输入的控制方式。这是一种**负反馈**调节方式。

通过安装在电机上的传感器，我们可以得到电机目前实际转动的角速度，通过这一角速度和目标速度的差值，决定我应该向电机提供的电流。

如果太快了，就给少一点；如果太慢了，就给多一点。

![image-20240427152318622](如何让我的机器人动起来——SLAM、导航与底盘\pid.png)



## 参考资料

[Nav2 — Nav2 1.0.0 documentation (ros.org)](https://navigation.ros.org/)

[动手学ROS2 (fishros.com)](https://fishros.com/d2lros2/#/)

[视觉SLAM十四讲](https://github.com/gaoxiang12/slambook2)

[什么是 PID 控制器：工作原理及其应用_pid控制-CSDN博客](https://blog.csdn.net/qq_41415238/article/details/122079899)

[两轮差速小车运动学分析 - 古月居 (guyuehome.com)](https://www.guyuehome.com/33953)