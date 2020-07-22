# 吉林大学TARS-GO战队视觉代码JLURoboVision
---
## 介绍  
本代码是吉林大学TARS-GO战队Robomaster2020赛季步兵视觉算法，主要模块分为**装甲板识别**、**大风车能量机关识别**、**角度解算**、**相机驱动**及**串口/CAN通信**。  

---
## 目录
* [1. 功能介绍](#1功能介绍)
* [2. 效果展示](#2效果展示)
* [3. 依赖环境](#3依赖环境)
* [4. 整体框架](#4整体框架)
* [5. 实现方案](#5实现方案)
* [6. 通讯协议](#6通信协议)
* [7. 配置与调试](#7配置与调试)
* [8. 总结展望](#8总结展望)
---
## 1.功能介绍
|模块     |功能     |
| ------- | ------ |
|装甲板识别| 检测敌方机器人装甲板位置信息并识别其数字 |
|大风车能量机关识别| 检测待激活大风车扇叶目标位置信息 |
|角度解算| 根据上述位置信息解算目标相对枪管的yaw、pitch角度及距离 |
|相机驱动| 大恒相机SDK封装，实现相机参数控制及图像采集 |
|串口/CAN通信| 与下位机通信，传输机器人姿态信息及操作手反馈视觉的控制信息 |
---
## 2.效果展示
### 装甲板识别
装甲板识别采用基于OpenCV的传统算法实现装甲板位置检测，同时采用SVM实现装甲板数字识别。  
考虑战场实际情况，机器人可打击有效范围在1m~7m之间，在此范围内，本套算法**装甲板识别率达98%**，识别得到装甲板在图像中四个顶点、中心点的坐标信息。  
![图2.1 装甲板实时识别]( "title") 
在640*480图像分辨率下，**装甲板识别帧率可达340fps左右，引入ROI之后可达420fps**。但考虑到识别帧率对于电控机械延迟的饱和，取消引入ROI操作，以此避免引入ROI之后无法及时探测全局视野情况的问题，加快机器人自瞄响应。  
【图片】  
装甲板数字识别采用SVM，通过装甲板位置信息裁剪二值化后的装甲板图像并透射变换，投入训练好的SVM模型中识别，**数字识别准确率可达98%**。  
【图片】  
### 大风车能量机关识别
【图片】  
### 角度解算  
角度解算方面使用了两种解算方法分距离挡位运行。第一档使用P4P算法，第二档使用小孔成像原理的PinHole算法。  
此外还引入了相机-枪口的Y轴距离补偿及重力补偿。  
使用标定板测试，角度解算计算的距离误差在10%以内，角度基本与实际吻合。  
【图片】  

---
## 3.依赖环境
### 硬件设备
|硬件|型号|参数|
|---|---|---|
|运算平台|Manifold2-G|Tx2|
|相机|大恒相机MER-050|分辨率640*480 曝光值3000~5000|
|镜头|M0814-MP2|焦距8mm 光圈值4|
### 软件设备
|软件类型|型号|
|---|---|
|OS|Ubuntu 16.04/Ubuntu18.04|
|IDE|Qt Creator-4.5.2|
|Library|OpenCV-3.4.0|
|DRIVE|Galaxy SDK|
---
## 4.整体框架

---
## 5.实现方案  
### 装甲板识别  
装甲板识别使用的了基于OpenCV的传统方法，实现检测识别的中心思想是使用装甲板两侧灯条拟合装甲板。  
主要步骤分为：图像预处理、灯条检测、装甲板匹配、装甲板数字识别及最终的目标装甲板选择。  
#### 图像预处理  
为检测红/蓝灯条，需要进行颜色提取。颜色提取基本思路有BGR、HSV、通道相减法。  
然而，前两种方法由于需要遍历所有像素点，耗时较长，因此我们选择了**通道相减法**进行颜色提取。  
其原理是在**低曝光**（3000~5000）情况下，蓝色灯条区域的B通道值要远高于R通道值，使用B通道减去R通道再二值化，能提取出蓝色灯条区域，反之亦然。  
此外，我们还对颜色提取二值图进行一次掩膜大小3*3，形状MORPH_ELLIPSE的膨胀操作，用于图像降噪及灯条区域的闭合。  
【图片】
#### 灯条检测  
灯条检测主要是先对预处理后的二值图找轮廓（findContours），  
然后对初筛（面积）后的轮廓进行拟合椭圆（fitEllipse），  
使用得到的旋转矩形（RotatedRect）构造灯条实例（LightBar），  
在筛除偏移角过大的灯条后依据灯条中心从左往右排序。  
【图片】  
#### 装甲板匹配  
分析装甲板特征可知，装甲板由两个长度相等互相平行的侧面灯条构成，  
因此我们对检测到的灯条进行两两匹配，  
通过判断两个灯条之间的位置信息：角度差大小、错位角大小、灯条长度差比率和X,Y方向投影差比率，  
从而分辨该装甲板是否为合适的装甲板（isSuitableArmor），  
然后将所有判断为合适的装甲板放入预选装甲板数组向量中。  
同时，为了消除“游离灯条”导致的误装甲板，我们针对此种情况编写了eraseErrorRepeatArmor函数，专门用于检测并删除错误装甲板。  
【图片】
匹配好装甲板后，利用装甲板的顶点在原图的二值图（原图的灰度二值图）中剪切装甲板图，  
使用透射变换将装甲板图变换为SVM模型所需的Size，随后投入SVM识别装甲板数字。  
【图片】
对上述各项装甲板信息（顶点中心点坐标与枪口锚点距离、面积大小、装甲板数字及其是否与操作手设定匹配）进行加权求和，  
从而获取最佳打击装甲板作为最终的目标装甲板。  
【图片】

---
### 大风车识别

### 角度解算  
角度解算部分使用了两种模型解算枪管直指向目标装甲板所需旋转的yaw和pitch角。  
第一个是P4P解算，第二个是PinHole解算。  
首先回顾一下相机成像原理，其成像原理公式如下：  
$ s \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix} \begin{bmatrix} r_{11} & r_{12} & r_{13} & t_1 \\ r_{21} & r_{22} & r_{23} & t_2 \\ r_{31} & r_{32} & r_{33} & t_3 \end{bmatrix} \begin{bmatrix} X \\ Y \\ Z \\ 1 \end{bmatrix}$
其中：  
物体成像平面坐标：  
$ \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} $  
相机内参矩阵：  
\begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}  
旋转向量和平移向量：  
\begin{bmatrix} r_{11} & r_{12} & r_{13} & t_1 \\ r_{21} & r_{22} & r_{23} & t_2 \\ r_{31} & r_{32} & r_{33} & t_3 \end{bmatrix}  
物体世界坐标：  
\begin{bmatrix} X \\ Y \\ Z \\ 1 \end{bmatrix}

---
## 6.通讯协议

---
## 7.配置与调试

---
## 8.总结展望


#### 软件架构
软件架构说明


#### 安装教程

1.  xxxx
2.  xxxx
3.  xxxx

#### 使用说明

1.  xxxx
2.  xxxx
3.  xxxx

#### 参与贡献

1.  Fork 本仓库
2.  新建 Feat_xxx 分支
3.  提交代码
4.  新建 Pull Request


#### 码云特技

1.  使用 Readme\_XXX.md 来支持不同的语言，例如 Readme\_en.md, Readme\_zh.md
2.  码云官方博客 [blog.gitee.com](https://blog.gitee.com)
3.  你可以 [https://gitee.com/explore](https://gitee.com/explore) 这个地址来了解码云上的优秀开源项目
4.  [GVP](https://gitee.com/gvp) 全称是码云最有价值开源项目，是码云综合评定出的优秀开源项目
5.  码云官方提供的使用手册 [https://gitee.com/help](https://gitee.com/help)
6.  码云封面人物是一档用来展示码云会员风采的栏目 [https://gitee.com/gitee-stars/](https://gitee.com/gitee-stars/)
