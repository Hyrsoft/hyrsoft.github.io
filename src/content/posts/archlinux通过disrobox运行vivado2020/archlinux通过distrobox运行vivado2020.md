---
title: Arch Linux通过Distrobox运行Vivado2020
published: 2026-01-26
description: '摆脱Ubuntu虚拟机难用的图形化界面和繁琐的配置'
image: './image.png'
tags: [ZYNQ, FPGA, Distrobox, Vivado]
category: 'FPGA'
draft: false 
lang: ''
---
对于**Arch Linux**运行Vivado开发Zynq来说，一般会考虑**KVM虚拟机**安装对应版本的Ubuntu，但是抛开内存和性能损耗不谈，单单是USB设备挂载、NFS挂载等后续问题，就够麻烦了，而且老版本Ubuntu的GUI结合KVM难绷的图形性能（尤其是N卡，无法做到显卡直通和3D加速），对于使用Vivado这样的图形化界面来说非常难受。

这里选择使用 **Distrobox** 进行环境配置。它通过与宿主机共享内核和命名空间，让 Ubuntu 不再是隔离的虚拟系统，而是以“透明环境”的形式融入 Arch Linux。这种架构使 Vivado 可以直接调用宿主机的 NVIDIA 驱动进行硬件加速渲染，避免了 KVM 虚拟机中常见的界面卡顿和操作迟滞。同时，容器共享 `/dev` 设备节点和网络栈，使 JTAG 下载、Zynq 调试及 NFS 等网络服务无需额外配置即可使用，整体体验接近原生 Ubuntu LTS，大幅降低了环境折腾成本。

相比之下，**Docker 更偏向应用级容器**，默认权限受限，对 GUI 程序和硬件访问并不友好。在 Vivado 这类重度依赖图形界面与外设的开发场景下，Docker 往往需要额外处理显示转发、设备映射和权限问题，配置复杂且稳定性不足，更适合作为自动化构建或批处理工具，而非日常交互式开发环境。

---

**使用环境**：Arch Linux、KED桌面、Wayland、Nvidia独立显卡

既然Wayland和n卡都能跑通，其他环境只会更方便

## 一、通过Distrobox安装Ubuntu20.04

#### 安装和配置Distrobox

代理相关的不再赘述

优先选择podman

```bash
sudo pacman -S podman distrobox
```

创建和启动容器，并初始化

```bash
distrobox create -n fpga-dev --image ubuntu:20.04

# 进入容器
distrobox enter fpga-dev
```

distrobox容器会继承宿主机的shell设置（比如oh-my-zsh）和/dev，可谓相当方便

进入ubuntu容器后，安装需要的软件

```bash
# 更新源并安装 Vivado 必需的依赖和常用工具
sudo apt update
sudo apt install -y libtinfo5 libncurses5 libx11-6 libxrender1 libxtst6 libxi6 libglib2.0-0 build-essential bc locales git vim wget curl libxft2 libfontconfig1 libdbus-1-3

# 修正 Ubuntu 的 dash 问题（Petalinux 强依赖 bash）
sudo dpkg-reconfigure dash # 弹窗选择 No
```

## 二、下载和安装Vivado2020

[下载网址](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/archive.html)，下载时注意关闭代理，不然四五十GB挺肉疼的

![截屏2026-01-25 14.22.05](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260125142210713.png)

#### 解压与安装

```bash
tar -xzf Xilinx_Unified_2020.2.tar.gz

cd Xilinx_Unified_2020.2_1118_1232 

# 注意是在宿主机桌面distrobox的ubuntu终端里执行，不能是ssh的终端里
sudo ./xsetup
```

根据引导一步步安装

![截屏2026-01-25 20.52.59](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260125235105699.png)

选择Vitis

![截屏2026-01-25 20.53.12](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260125235116364.png)

对于zynq7010/7020的话就是选Zynq-7000，我这里顺带安装一下Artix-7，未来可能会用

![截屏2026-01-25 20.54.13](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260125235127997.png)

## 三、启动Vivado

实测直接启动后会白屏，搜索得知需要添加一个环境变量

[Linux下安装Vivado打开后空白屏的解决方案](https://blog.csdn.net/weixin_52027058/article/details/128279806)

```bash
# 可以添加到宿主机的~/.bashrc或者~/.zshrc里
export _JAVA_AWT_WM_NONREPARENTING=1
```

![截屏2026-01-25 23.46.05](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260125235246943.png)

```bash
# 根据安装时的具体位置
source /tools/Xilinx/Vivado/2022.2/settings64.sh

# 启动Vivado的图形化界面
vivado
```

结果如图：

![48688AD08028BE650FBA02D648B15086](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20260125235750134.png)
