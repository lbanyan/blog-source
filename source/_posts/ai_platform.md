---
title: AI平台化建设概要
date: 2017-09-09 10:36
tags:
    - AI平台
---

### 平台架构
![平台架构](/img/ai/ai_frame.png)

<!--more-->

- 可视化平台，用于用户与数据平台交互，管理计算资源，模型训练及评估和分析管理；
- AI平台以VM的方式提供计算资源。
- 预处理后的数据存储于VM本地，用户可登录VM再次进行数据的预处理、特征转换以及模型训练，模型训练也可以在可视化平台界面操作完成。
- 用户创建机器学习项目时，默认提供一台VM作为计算资源，此VM可预装Tensorflow、MXnet或Caffe环境，来灵活满足用户需要，同时在资源允许的情况下，可以提供云物理机和GPU处理能力。

### 业务流程
![](/img/ai/ai_data_process.png)

### 所需算法
![算法](/img/ai/ai_algorithm.png)
- 原始数据来自于数据平台；
- 数据预处理配置由用户在AI可视化平台配置，AI平台通过与数据中心交互获取到预处理后的特定格式数据；
- 其他部分图中展示的算法是目前已经在用的算法，未来还应支撑到更多的算法。

### AI平台功能
- AI可视化平台
 - 用户预处理数据配置管理
 - 模型训练管理
 - 模型评估及分析管理
- 数据通道管理
- 计算资源
 - 计算硬件环境管理
 - 模型训练环境定制

### 预处理后的数据格式
效仿AWS标准：
- csv标准格式，utf-8编码
- 每行表示一条完整的训练数据
- 首行为各列数据的名称
- 数据应是扁平化的，即一列数据中不允许出现数组、Map等等复杂结构

例如：
``` text
customerId,jobId,education,housing,loan,campaign,duration,willRespondToCampaign
1,3,basic.4y,no,no,1,261,0
2,1,high.school,no,no,22,149,0
3,1,high.school,yes,no,65,226,1
4,2,basic.6y,no,no,1,151,0
```
前期可以采用这种严格要求，后期逐渐优化到可支持到数据的复杂结构，这样才会更加灵活。
更多要求请参考[Understanding the Data Format for Amazon ML](http://docs.aws.amazon.com/zh_cn/machine-learning/latest/dg/understanding-the-data-format-for-amazon-ml.html "Understanding the Data Format for Amazon ML")

### 补充
- 计算资源
提供基于预装Tensorflow、MXnet或Caffe，以及可公用的算法的大容量基础镜像，以便创建VM计算资源；对有特殊需求的，也可以创建云物理机或GPU计算资源。


