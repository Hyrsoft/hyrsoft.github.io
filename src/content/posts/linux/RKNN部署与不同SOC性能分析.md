---
title: RKNN 部署与不同 SOC 性能分析
published: 2026-02-26
description: '记录 RKNN 的部署流程，测试 YOLOv5 模型在不同硬件平台上的性能表现，分析适用场景。'
image: './images/RKNN部署与不同SOC性能分析_1.png'
tags: [rknn, 异构计算, C++, yolo, rv1106, rk3588]
category: 'linux'
draft: false 
lang: ''
---
## 一、概述

记录 RKNN 的部署流程，测试 YOLOv5 模型在不同硬件平台上的性能表现，分析适用场景。



需要三个仓库：

- [yolov5](https://github.com/airockchip/yolov5)

  一个极其流行的单阶段目标检测算法实现库。它是**模型的来源**。在这里获取原始的权重文件（`.pt`），并利用其中的 `export.py` 脚本，配合 `--rknpu` 参数，导出一个专门为瑞芯微 NPU 结构优化过的中间格式（`.onnx`）。

- [rknn-toolkit2](https://github.com/rockchip-linux/rknn-toolkit2)

  瑞芯微为旗下 NPU 芯片提供的软件开发套件（SDK）。它是**转换的核心工具**。它提供 C 或 Python 接口，负责把 `.onnx` 模型转换成板子能跑的 `.rknn` 格式。此外，它还承担了**模型量化**（将浮点数转为 `int8`）和**精度评估**的重要任务。

- [rknn_model_zoo](https://github.com/airockchip/rknn_model_zoo)

  瑞芯微官方提供的各种主流算法（如 YOLO、MobileNet 等）的部署示例集合。**测试工程模板**。它提供了经过优化的 C++ 源码（如 `rknn_yolov5_demo`）和交叉编译脚本。最终在板子上运行的那个“测速程序”，就是从这个仓库里编译出来的。

两个python虚拟环境

- yolov5 环境

	安装了原生 PyTorch 和 YOLOv5 所需的高版本依赖。**负责模型导出**。在这个环境下，运行 `python export.py`，把 `.pt` 变成 `.onnx`。一旦拿到 `.onnx`，这个环境的任务就完成了。
	
- RKNN-toolkit2 环境

	安装了特定版本的 `onnx`和瑞芯微私有的 `rknn-toolkit2` 软件包。**负责模型量化与转换**。在这个环境下，运行转换脚本，把上一步得到的 `.onnx` 喂进去，经过 `int8` 量化，最后得到出 `.rknn` 文件。

![image-20260226230207856](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260226230208030.png)

---



## 二、环境搭建

克隆上述三个仓库到`~/projects/rknn/`

**RKNN-toolkit2 环境搭建**

```bash
conda create -n RKNN-Toolkit2 python=3.9
conda activate RKNN-Toolkit2

# 克隆工具链
git clone https://github.com/airockchip/rknn-toolkit2.git
cd rknn-toolkit2

# 安装依赖（注意根据 Python 版本选择对应的 requirements 文件）
pip install -r rknn-toolkit2/packages/x86_64/requirements_cp39-2.3.2.txt -i https://pypi.tuna.tsinghua.edu.cn/simple/

# 安装 RKNN-Toolkit2 软件包
pip install rknn-toolkit2/packages/x86_64/rknn_toolkit2-2.3.2-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
```



**Yolov5 环境搭建**

```bash
# 克隆仓库
git clone https://github.com/airockchip/yolov5.git

cd yolov5

# 创建虚拟环境
conda create -n yolov5 python=3.9
# 进入虚拟环境并安装依赖库
conda activate yolov5
pip install -r requirements.txt
```

---



## 三、转换模型：ONNX to RKNN

**获取导出 ONNX 文件**

（yolov5 conda 环境下）

```bash
cd yolov5
python export.py --rknpu --weight yolov5s.pt
```

得到 `yolov5s.pt`、`yolov5s.onnx`和`RK_anchors.txt`



**获取 RKNN 文件**

（在RKNN-toolkit2 conda 环境下）

```bash
cd rknn_model_zoo/examples/yolov5/python

# 指定 onnx 模型的位置和 target（rv1106）
python3 convert.py ~/projects/rknn/yolov5/yolov5s.onnx rv1106
```

转换为RKNN文件`examples/yolov5/model/yolov5.rknn`



## 四、编译测试程序

在 `~/projects/rknn/rknn_model_zoo/examples/yolov5/cpp`

```bash
# 导入编译工具链路径
export GCC_COMPILER=/home/hao/projects/luckfox-pico/tools/linux/toolchain/arm-rockchip830-linux-uclibcgnueabihf/bin/arm-rockchip830-linux-uclibcgnueabihf

./build-linux.sh -t rv1106 -a armv7l -d yolov5 
```

编译并完成安装，将`rknn_model_zoo/install/rv1106_linux_armv7l/rknn_yolov5_demo`通过 scp 协议传输到开发板上，执行程序

```bash
./rknn_yolov5_demo model/yolov5.rknn model/bus.jpg
```

（原版代码只会执行一次，修改了demo的源码，让它连续执行10次并获取精确的 NPU 推理时常，计算结果后打印出来，修改后的源码在 https://github.com/Hyrsoft/rknn_model_zoo/tree/yolov5_benchmark/examples/yolov5/cpp）

运行结果：

```bash
[root@luckfox rknn_yolov5_demo]# ./rknn_yolov5_demo model/yolov5.rknn model/bus.jpg
load lable ./model/coco_80_labels_list.txt
model input num: 1, output num: 3
input tensors:
  index=0, name=images, n_dims=4, dims=[1, 640, 640, 3], n_elems=1228800, size=1228800, fmt=NHWC, type=INT8, qnt_type=AFFINE, zp=-128, scale=0.003922
output tensors:
  index=0, name=output0, n_dims=4, dims=[1, 255, 80, 80], n_elems=1632000, size=1632000, fmt=NCHW, type=INT8, qnt_type=AFFINE, zp=-128, scale=0.003922
  index=1, name=367, n_dims=4, dims=[1, 255, 40, 40], n_elems=408000, size=408000, fmt=NCHW, type=INT8, qnt_type=AFFINE, zp=-128, scale=0.003922
  index=2, name=369, n_dims=4, dims=[1, 255, 20, 20], n_elems=102000, size=102000, fmt=NCHW, type=INT8, qnt_type=AFFINE, zp=-128, scale=0.003922
input_attrs[0].size_with_stride=1228800
model is NHWC input fmt
model input height=640, width=640, channel=3
origin size=640x640 crop size=640x640
input image: 640 x 640, subsampling: 4:2:0, colorspace: YCbCr, orientation: 1
scale=1.000000 dst_box=(0 0 639 639) allow_slight_change=1 _left_offset=0 _top_offset=0 padding_w=0 padding_h=0
rga_api version 1.10.1_[0]
rknn_run
========================================
NPU Inference time:
First run time: 85.209 ms
Average time of 10 runs: 86.624 ms
Estimated FPS: 11.54
========================================
person @ (209 242 283 516) 0.828
person @ (474 230 561 522) 0.798
person @ (114 235 207 543) 0.796
bus @ (93 133 549 462) 0.787
person @ (78 330 122 519) 0.408
write_image path: out.png width=640 height=640 channel=3 data=0xa5a31000
```

推理结果：

![out](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260226211359529.png)

---



## 五、结果分析



### 瑞芯微主流 SoC NPU 性能实测数据汇总表

除了 RV1106G3 的数据为本文章实测外，其他数据由触觉智能的文章提供。

**测试条件**：YOLOv5s-640-640 模型，INT8 量化，640*640 分辨率测试图片。

| **芯片型号** | **核心硬件参数**               | **首次运行耗时 (ms)** | **连续10次平均耗时 (ms)** | **帧率估算 (FPS)** |
| ------------ | ------------------------------ | --------------------- | ------------------------- | ------------------ |
| **RV1106G3** | 单核 A7 + 1.0 TOPS NPU         | 85.209                | 86.624                    | **11.54**          |
| **RK3562**   | 四核 A53 + 1.0 TOPS NPU        | 66.165                | 51.885                    | **19.3**           |
| **RV1126B**  | 四核 A53 + 3.0 TOPS NPU        | 26.897                | 26.0425                   | **38.4**           |
| **RK3576**   | 4核A72 + 4核A53 + 6.0 TOPS NPU | 55.395                | 23.7342                   | **42.1**           |
| **RK3588**   | 4核A76 + 4核A55 + 6.0 TOPS NPU | 30.887                | 21.2581                   | **47.0**           |

![2a1856faff4937fdaca5ed6c01602077](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260226223055099.png)

------

###  实验结果分析与硬件适配性评估

通过对瑞芯微（Rockchip）系列 SoC 的推理性能实测发现，其 NPU 推理效率遵循 RK3588 > RK3576 > RV1126B > RK3562 > RV1106G3 的梯度分布，其中具备 6 TOPS 算力的 RK3588 与 RK3576 以超过 40 FPS 的表现领跑高实时性场景；作为对比，主打超低功耗与成本优势的 **RV1106G3**（单核 A7 + 1.0 TOPS NPU）在处理 640*640 大分辨率模型时实现了 **11.54 FPS** 的实测帧率，且展现出极佳的时延确定性，其首次运行与平均耗时波动显著低于多核架构芯片，使其在智能门铃、低功耗 IPC 等对响应稳定性要求较高、且成本敏感的垂直场景中，展现出比四核 A53 架构的 RK3562 更为突出的性价比优势。



## 参考链接

- https://mp.weixin.qq.com/s/8cPELPb-4YGDo0fZ_o--Aw
- https://wiki.luckfox.com/zh/Luckfox-Pico-Ultra/RKNN

