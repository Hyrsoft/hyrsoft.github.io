---
title: 使用pyocd进行mcu的烧录和调试
published: 2025-04-25
description: ''
image: ''
tags: [pyocd mcu]
category: 'tool'
draft: false 
lang: ''
---
# 简介

pyOCD 是一个开源 Python 软件包，用于多种受支持的硬件调试器（DAP-Link、J-Link、ST-Link,CMSIS-DAP v1(HID)、CMSIS-DAP v2(WinUSB)、SEGGER J-Link、ST-LINK v2和ST-LINK v3）下编程和调试Arm Cortex-M微控制器。它是完全跨平台的，并支持Linux，macOS和Windows。它内置支持多达70种流行的MCU。通过使用CMSIS-Pack，几乎支持市场上的所有Cortex-M设备。pyOCD还可以作为GDB Service配合GDB调试芯片。

使用环境：Ubuntu22.04

## 一、安装pyocd

利用pip进行安装，确保当前环境中python版本大于3.6

安装pip

```bash
sudo apt update
sudo apt install python3-pip
```

更新pip

```bash
python3 -m pip install --upgrade --force pip
```

安装pyocd

```bash
python3 -m pip install -U pyocd
```

安装成功后重启Ubuntu，查看是否安装成功

```bash
pyocd -V # 注意是大写V
```

## 二、连接烧录设备

列出当前连接上的烧录设备

```bash
pyocd list

#   Probe/Board    Unique ID                  Target  
--------------------------------------------------------
  0   STM32 STLink   00380043320000054E575359   n/a    
```

## 三、添加对MCU的支持

列出当前支持的所有mcu：

```bash
pyocd list -t

Name                      Vendor                   Part Number                  Families   Source   
------------------------------------------------------------------------------------------------------
  air001                    AirM2M                   Air001                                  builtin  
  air32f103xb               AirM2M                   Air32F103xB                             builtin  
  air32f103xc               AirM2M                   Air32F103xC                             builtin  
  air32f103xe               AirM2M                   Air32F103xE                             builtin  
  air32f103xg               AirM2M                   Air32F103xG                             builtin  
  air32f103xp               AirM2M                   Air32F103xP                             builtin  
  cc3220sf                  Texas Instruments        CC3220SF                                builtin  
......
```

查询对指定型号的支持：

```bash
pyocd list -t | grep stm32*

stm32f051                 STMicroelectronics       STM32F051     builtin  
stm32f103rc               STMicroelectronics       STM32F103RC   builtin  
  
......
```

### 1、查询并添加对某款MCU的支持

更新包索引

```bash
pyocd pack update
```

查找mspm0g3507

```bash
pyocd pack -f mspm0g3507

Part         Vendor              Pack                          Version   Installed  
--------------------------------------------------------------------------------------
  MSPM0G3507   Texas Instruments   TexasInstruments.MSPM0G_DFP   1.2.1     False     
```

如果查询到了就可以直接安装

```bash
pyocd pack -i mspm0g3507

Downloading packs (press Control-C to cancel):
    TexasInstruments.MSPM0G_DFP.1.2.1
Downloading descriptors (001/001)
```

此时再执行`pyocd pack -f mspm0g3507`，`Installed `就会变成True，说明已经支持

### 2、手动添加对某款MCU的支持

有时候pyocd没有初始支持对某款mcu，查询也查询不到，可以手动下载对应的pack包并进行安装

比如默认就找不到stm32f407zgt6：

```bash
pyocd pack -i stm32f407zgt6

0000550 W No matching devices. Please make sure the pack index is up to date. [pack_cmd]
Downloading packs (press Control-C to cancel):
```

可以从Keil官网下载对应的DFP包，在对目标mcu进行操作的时候作为参数来添加进去

```bash
pyocd list -t --pack ./Keil.STM32F4xx_DFP.3.0.0.pack
```

这样就可以列出包中支持的mcu

**可以把需要的pack放置到指定目录，在对需要的单片机进行操作时，加入`--pack`参数**



## 三、擦除和烧录MCU

### 烧录

```bash
pyocd flash --target stm32f407zgtx ./build/demo.hex --pack ~/Keil.STM32F4xx_DFP.3.0.0.pack 

pyocd flash --target mspm0g3507 ./build/empty.hex 
```

如果需要操作的mcu已有支持，不需要加`--pack`参数

**注意：**烧录的单片机是stm32f407zgt6，但是下载的pack中对应的型号是stm32f407zgtx，可以通过`pyocd list -t`来查看。

### VScode快捷任务

`.vscode/tasks.json`

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Flash",
            "type": "process",
            "command": "pyocd",
            "args": [
              "flash",
              "--target",
              "stm32f407zgtx",
              "--pack",
              "~/Keil.STM32F4xx_DFP.3.0.0.pack",
              "./build/demo.hex"
            ],
            "problemMatcher": [],
            "group": {
              "kind": "build",
              "isDefault": true
            },
            "presentation": {
              "reveal": "always",
              "panel": "dedicated",
              "clear": true
            },
            "detail": "Flash demo.hex to STM32 via ST-Link (pyocd)"
          }
    ]
}
```

## 四、调试MCU

**注意：**可执行文件使用`.elf`

调试的时候需要下载VScode插件**Cortex Debug**

`.vscode/launch.json`

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug",
            "cwd": "${workspaceFolder}",
            "executable": "./build/demo.elf",
            "request": "launch",
            "type": "cortex-debug",
            "runToEntryPoint": "main",
            "servertype": "pyocd",
            "targetId": "stm32f407zgtx",
            "cmsisPack": "~/Keil.STM32F4xx_DFP.3.0.0.pack"
        }
    ]
}
```

调试结束后，确定gdb进程关闭，否则下次进入调试会报错

