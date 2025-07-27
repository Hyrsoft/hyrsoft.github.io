---
title: python脚本从嘉立创导入元器件到kicad
date: 2025-07-27 21:43:55
tags:
---
## 一、Windows中miniconda的安装与使用

**安装**

[官方安装指南](https://www.anaconda.com/docs/getting-started/miniconda/install#windows-powershell)，以此执行一下三条指令

```powershell
wget "https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe" -outfile ".\miniconda.exe"
Start-Process -FilePath ".\miniconda.exe" -ArgumentList "/S" -Wait
del .\miniconda.exe
```

**常用命令**

列出当前已有虚拟环境 `conda env list`



**创建虚拟环境**

```powershell
conda create -n easyeda python=3.10 # 创建名为easyeda的虚拟环境，指定python版本

# 激活对应虚拟环境
conda activate easyeda

# 离开当前虚拟环境
conda deactivate
```

## 二、easyeda2kicad脚本的安装与使用

[easyeda2kicad仓库](https://github.com/uPesy/easyeda2kicad.py)

在对应虚拟环境中执行`pip install easyeda2kicad`即可安装

**基于元器件客编号来导入元器件**

![Snipaste_2025-07-14_22-47-54](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1202507142248226.png)

找到立创商城里需要的元件的编号，在虚拟环境中把它导入到名为`easyeda2kicad`的元件库中

```powershell
easyeda2kicad --full --lcsc_id=C32710423


# 显示创建的元件名与库的地址
[INFO] Created Kicad symbol for ID : C32710423
       Symbol name : LCKFB-LSPI-SKYSTAR-STM32F407VGT6-PRO
       Library path : C:\Users\hao\Documents\Kicad\easyeda2kicad/easyeda2kicad.kicad_sym
[INFO] Created Kicad footprint for ID: C32710423
       Footprint name: COMM-TH_LCKFB-LSPI-SKYSTAR-STM32F407VGT6-PRO
       Footprint path: C:\Users\hao\Documents\Kicad\easyeda2kicad/easyeda2kicad.pretty\COMM-TH_LCKFB-LSPI-SKYSTAR-STM32F407VGT6-PRO.kicad_mod
[WARNING] No 3D model available for this component
```

**导入easyeda对应的符号库和封装库**

在kicad开始界面的设置中的符号库与封装库中，分别导入对应文件和路径

![Snipaste_2025-07-14_22-51-38](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1202507142254049.png)

![Snipaste_2025-07-14_22-52-04](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1202507142254068.png)

**在kicad中找到导入的元件**

![Snipaste_2025-07-14_22-54-57](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1202507142255980.png)