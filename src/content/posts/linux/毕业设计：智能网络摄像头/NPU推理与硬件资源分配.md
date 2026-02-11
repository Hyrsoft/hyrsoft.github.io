---
title: 毕设日志（四）NPU 推理与硬件资源分配
published: 2026-02-12
description: '添加 NPU 推理模块时触到了硬件性能瓶颈'
image: './images/NPU推理与硬件资源分配.png'
tags: [rknn, 异构计算, c++, rockit]
category: 'linux'
draft: false 
lang: ''
---
经过漫长的摸索和Debug，我意识到在 RV1106G3 这个单核A7 + 256MB DDR3 的嵌入式设备上，再精妙的设计也要向现实妥协：之前设想的异构并行逻辑，受限于 DDR3 的带宽，导致 NPU 获取图像超时，产生运行时报错。只能退而求其次，先用最原始的串行逻辑，就算显示帧率感人，CPU 占用高，但先把基础功能实现了，再谈性能优化。

[Luckfox 的 RKMPI 例程](https://wiki.luckfox.com/zh/Luckfox-Pico-Zero/MPI)中提供了 YOLOv5 和 Retainface 两种示例，虽然都是 NPU 推理，但是硬件资源分配和视频流处理的逻辑大不相同，其中，YOLOv5 推理对性能要求是比 Reatinface 高很多的。

::github{repo="LuckfoxTECH/luckfox_pico_rkmpi_example"}

---


## 一、三种硬件资源（VI、VPSS、VENC）分配方式



为了完成**采集图像给 NPU 推理，把推理结果标注到视频流上，编码后进行流媒体分发**这个基本流程，在硬件资源分配上，一共有三种方式：

### 1. VI 通道分流

VI 通道分为 ch0 和 ch1，一路输出 NV12 给 VENC 编码，一路输出 NV12 经由格式转换和缩放后给NPU推理使用。硬件流水线，效率最高，CPU 压力小，但是多硬件模块并行访问内存，内存带宽压力最大。运行 Retainface 这个轻量级模型还行。

在使用 VI 通道分流时，就算没有加入 NPU 推理，也会有 VI 通道采集报错，在使用 VPSS 通道分流的基础上，完成了 IPC 的基础功能，后续就没有再考虑过 VI 通道分流。

Retainface 例程使用的是 VI 通道分流

![retainface](https://wiki.luckfox.com/img/RV1106/RKNN/020.jpg)

### 2. VPSS 通道分流



VI 通道只保留一个，VPSS 通道分为两个，一个给 VENC 编码，一个缩放到模型需要的大小（例如 YOLOv5 模型需要的是 640x640 ），再经由格式转换后给 NPU 推理。

使用 VPSS 通道分流时，VPSS 通道和图像处理（RGA 或者 opencv-mobile）能输出正确的图像，但仍有NPU 获取图像超时。

![VPSS 通道分流](https://wiki.luckfox.com/img/RV1106/RKNN/008.jpg)

### 3. 不分流，串行逻辑

VI 通道和 VPSS 通道只保留一个，不使用硬件绑定（自动流水线），而是软件手动进行流转。先格式转换 + 缩放，NPU 推理后进行标注，再进行编码和流媒体分发。CPU 压力最大，效率（帧率）最低，但是内存带宽压力小。

YOLOv5 使用的是串行逻辑

![yolov5](https://wiki.luckfox.com/img/RV1106/RKNN/017.jpg)



## 二、AIPC 软件设计

::gitHub{repo="Hyrsoft/aipc"}

理想情况下，是使用 VI 通道分流，或者 VPSS 通道分流，尽可能利用 RV1106 的硬件处理单元，降低 CPU 压力，但是这条路让我踩了无数坑，不是 VI 采集失败就是 NPU 推理失败，基本都是硬件处理单元压力过大、内存带宽压力过大导致的，只能退而求其次。

对于我毕设准备实现的 YOLOv5 和 Retainface 两个模型来说，它们对硬件资源和内存的需求各不相同，当前我通过一刀切的方式实现了“跑起来”这个基础需求：

- 纯 IPC 模式：不进行硬件分流，最大化利用硬件资源进行采集和编解码，实现 1080p 推流
- AI 推理模式：不进行硬件分流，串行逻辑，在个位数帧率下实现了 YOLOv5 推理 + 标注 + 推流

应该是设计一个冷切换的硬件资源分配策略（初始化系统时调用，切换模型时，对硬件资源进行重新配置），以满足不同模型对硬件资源的需求，也方便后续拓展和性能调优。