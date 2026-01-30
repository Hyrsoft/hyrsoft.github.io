---
title: rkdeveloptool
published: 2025-11-21
description: 'rkdeveloptool的基础使用'
image: ''
tags: [linux, rockchip]
category: 'tools'
draft: false 
lang: ''
---
rk开源的烧录工具，可以用于烧录整个img镜像，例如armbian的镜像

::github{repo="rockchip-linux/rkdeveloptool"}

建议自行编译最新版本的，注意配置udev规则`99-rockchip.rules`（GitHub仓库里下载）

前提：设备进入maskrom模式

```bash
sudo rkdeveloptool ld
```

列出进入maskrom模式的设备

### 一、初始化Loader

初始化DRAM，刷入类似miniloader.bin的文件

包含了一段 SPL (Secondary Program Loader) 代码，它的主要职责就是**初始化内存时序**

```bash
sudo ./rkdeveloptool db rk35xx_spl_loader.bin
```

### 二、刷入镜像

```bash
sudo ./rkdeveloptool wl 0 system_image.img
```

### 三、重启设备

```bash
sudo ./rkdeveloptool rd
```

