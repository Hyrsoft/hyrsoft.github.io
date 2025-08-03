---
layout: pyocd
title: 使用pyocd调用FLM格式的外部下载算法
date: 2025-08-03 22:08:05
tags: pyocd/openocd
categories: MCU/单片机/微控制器
---
# pyocd 调用 FLM 格式的外部下载算法

记录如何通过 `pyocd` 调用 Keil 使用的 `.FLM` 格式外部下载算法，将程序烧录到 STM32H7 系列微控制器的 QSPI Flash 中。

### 项目背景

| 项目         | 内容                                                         |
| ------------ | ------------------------------------------------------------ |
| **开发板**   | [反客STM32H750XHB6](https://item.taobao.com/item.htm?id=838568127129&pisk=gIi_MI2qJ8VFHG385oJFVo7BQuZfhp-yDtwxExINHlETliHZN-S4uF2fcWMJI58DSjgIMX4a6Sz4c-GztAS2IAJXcxH86oua_-hEGXbZ_mPqTnMoNPSZHmWi-bkRbc8g0iZgmodya3-rQAq0DXNEywfGvRy60GEYHlqda56wy3-rIv6aBLkp4mlDc0oAkihYWyCLIWeTH-F9dyezHoITkGQL9W2YDoFOWyBLn81ADSF9d9eYBNQADPUd9-yVBoExMyHLt-EYHjEvdvF3HDUHAJJ41A9WHUubK3YovWsADDwpi5M1Cg2tfR98aANBwdoCqPN-BWsv6ibG95e-2B7gFPGbJDkVUiPrB4c0efC9aPDxRXwsTBbQd4liVknJtiGjEDcQpbKf45HnU4ziXESzPlP_lcH61_NKfXigXf8AVjls7XaQ0EjYXl2Ssln9T1Zjpczozyf6TSurLcUIwBSIirgo3-G6fGEEzf08r2AfP73rEkszcgP5F8QX2kSbd7JBdZbcEQOznV0AZ0zTKJ5ydp1_vPe3d7JBdZb0WJ2e3p9C1kC..&spm=tbpc.boughtlist.suborder_itemtitle.1.3c6a2e8dVW0dPQ&sku_properties=-1%3A-1) |
| **参考工程** | [Peakors/STM32H750XBH6_Template](https://github.com/Peakors/STM32H750XBH6_Template) |
| **下载算法** | `FK750M6_XBH6_V0.FLM` (由开发板资料提供)                     |
| **测试环境** | Fedora 42 (经验可跨平台通用)                                 |

![Snipaste_2025-08-03_22-05-27](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20250803220535715.png)

### 前置依赖

确保已安装以下工具链：

- `pyocd`
- `cmake`
- `ninja` 或 `make`
- `arm-none-eabi-gcc`

### 编译流程

```bash
# 生成 Ninja/Make 构建文件
cmake -S . -B build -G Ninja

# 执行编译
cmake --build build/
```

### 原理说明

通过在项目根目录创建 `pyocd_user.py` 脚本，可以自定义 pyOCD 的连接行为。pyOCD 在启动时会自动检测并加载该文件。

我们在脚本中定义一个 `will_connect` 函数，该函数会在 pyOCD 连接目标板之前被调用。通过这个函数，我们可以向 pyOCD 的内存地图中添加外部 Flash 的信息（如 QSPI Flash），并指定其使用的 `.FLM` 下载算法。

这种机制使得 pyOCD 能够在不修改内置目标支持的情况下，扩展对外部存储器的烧录能力。

**1. 目录结构**

- 将下载算法文件 `FK750M6_XBH6_V0.FLM` 存放到项目下的 `.config` 文件夹中。
- 在项目根目录创建 `pyocd_user.py` 文件。

**2. `pyocd_user.py` 脚本**

```python
# pyocd_user.py

from pyocd.core.memory_map import FlashRegion
import logging

# --- 定义外部 QSPI Flash 的参数 ---
FLM_FILE = ".config/FK750M6_XBH6_V0.FLM" # 存放下载算法的相对路径
QSPI_FLASH_START = 0x90000000
QSPI_FLASH_SIZE = 32 * 1024 * 1024
QSPI_FLASH_BLOCKSIZE = 0x1000

def will_connect(board):
    """
    这是一个 pyOCD 代理函数，会在连接前被自动调用。
    
    """
    target = board.target
    
    # 定义外部 Flash 区域
    external_flash = FlashRegion(
        name="qspi_flash",
        start=QSPI_FLASH_START,
        length=QSPI_FLASH_SIZE,
        blocksize=QSPI_FLASH_BLOCKSIZE,
        is_boot_memory=False,
        flm=FLM_FILE
    )
    
    # 将其添加到内存地图
    target.memory_map.add_region(external_flash)
    
    logging.info(f"用户脚本：成功添加外部 QSPI Flash 区域 '{external_flash.name}'")
```

具体定义参考`/home/hao/.local/lib/python3.13/site-packages/pyocd/core/memory_map.py`（pyocd安装路径）

![Snipaste_2025-08-03_21-27-37](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20250803215751929.png)

### 烧录指令

使用以下指令将编译生成的 `.bin` 文件烧录到 QSPI Flash 的起始地址 `0x90000000`。

- `--target stm32h750xx`：指定目标芯片型号。
- `-O connect_mode=under-reset`：设置连接模式为复位下连接。
- `./build/STM32H750XBH6_Template.bin@0x90000000`：指定固件路径和烧录地址。

```bash
pyocd flash --target stm32h750xx -O connect_mode=under-reset ./build/STM32H750XBH6_Template.bin@0x90000000
```

### 日志参考

**1. 简洁版日志**

```bash
❯ pyocd flash --target stm32h750xx -O connect_mode=under-reset ./build/STM32H750XBH6_Template.bin@0x90000000
0000447 I Loading /home/hao/projects/stm32_projects/STM32H750XBH6/CMake/STM32H750XBH6_Template/build/STM32H750XBH6_Template.bin at 0x90000000 [load_cmd]
[==================================================] 100%
0002722 I Erased 65536 bytes (1 sector), programmed 65536 bytes (8 pages), skipped 0 bytes (0 pages) at 28.14 kB/s [loader]
```

**2. 详细版日志 (`-v` 参数)**

```bash
❯ pyocd flash --target stm32h750xx -v -O connect_mode=under-reset ./build/STM32H750XBH6_Template.bin@0x90000000
0000346 I Target type is stm32h750xx [board]
0000349 I 用户脚本：成功添加外部 QSPI Flash 区域 'qspi_flash' [pyocd_user]
0000349 I Asserting reset prior to connect [coresight_target]
0000352 I DP IDR = 0x6ba02477 (v2 rev6) [dap]
0000366 I CR: 0x0070003f [target_STM32H750xx]
0000370 I AHB-AP#0 IDR = 0x84770001 (AHB-AP var0 rev8) [discovery]
0000373 I AHB-AP#1 IDR = 0x84770001 (AHB-AP var0 rev8) [discovery]
0000376 I APB-AP#2 IDR = 0x54770002 (APB-AP var0 rev5) [discovery]
0000379 I AHB-AP#0 Class 0x1 ROM table #0 @ 0xe00fe000 (designer=020:ST part=450) [rom_table]
0000381 I [0]<e00ff000:ROM class=1 designer=43b:Arm part=4c7> [rom_table]
0000381 I   AHB-AP#0 Class 0x1 ROM table #1 @ 0xe00ff000 (designer=43b:Arm part=4c7) [rom_table]
0000383 I   [0]<e000e000:SCS v7-M class=14 designer=43b:Arm part=00c> [rom_table]
0000385 I   [1]<e0001000:DWT v7-M class=14 designer=43b:Arm part=002> [rom_table]
0000386 I   [2]<e0002000:FPB v7-M class=14 designer=43b:Arm part=00e> [rom_table]
0000387 I   [3]<e0000000:ITM v7-M class=14 designer=43b:Arm part=001> [rom_table]
0000389 I [1]<e0041000:ETM M7 class=9 designer=43b:Arm part=975 devtype=13 archid=4a13 devid=0:0:0> [rom_table]
0000391 I [2]<e0043000:CTI CS-400 class=9 designer=43b:Arm part=906 devtype=14 archid=0000 devid=40800:0:0> [rom_table]
0000393 I APB-AP#2 Class 0x1 ROM table #0 @ 0xe00e0000 (designer=020:ST part=450) [rom_table]
0000396 I [2]<e00e3000:SWO CS-400 class=9 designer=43b:Arm part=914 devtype=11 archid=0000 devid=ea0:0:0> [rom_table]
0000398 I [3]<e00e4000:Trace Funnel CS-400 class=9 designer=43b:Arm part=908 devtype=12 archid=0000 devid=32:0:0> [rom_table]
0000399 I [4]<e00e5000:TSGEN class=15 designer=43b:Arm part=101> [rom_table]
0000400 I [5]<e00f0000:ROM class=1 designer=020:ST part=001> [rom_table]
0000401 I   APB-AP#2 Class 0x1 ROM table #1 @ 0xe00f0000 (designer=020:ST part=001) [rom_table]
0000404 I   [0]<e00f1000:CTI CS-400 class=9 designer=43b:Arm part=906 devtype=14 archid=0000 devid=40800:0:0> [rom_table]
0000406 I   [2]<e00f3000:Trace Funnel CS-400 class=9 designer=43b:Arm part=908 devtype=12 archid=0000 devid=34:0:0> [rom_table]
0000408 I   [3]<e00f4000:ETF class=9 designer=43b:Arm part=961 devtype=32 archid=0000 devid=380:0:0> [rom_table]
0000410 I   [4]<e00f5000:TPIU CS-400 class=9 designer=43b:Arm part=912 devtype=11 archid=0000 devid=a0:0:0> [rom_table]
0000415 I CPU core #0: Cortex-M7 r1p1, v7.0-M architecture [cortex_m]
0000415 I   Extensions: [DSP, FPU, FPU_DP, FPU_V5, MPU] [cortex_m]
0000415 I   FPU present: FPv5-D16-M [cortex_m]
0000416 I 4 hardware watchpoints [dwt]
0000419 I 8 hardware breakpoints, 1 literal comparators [fpb]
0000431 I Deasserting reset post connect [coresight_target]
0000433 I Creating flash algo for region qspi_flash from: /home/hao/projects/stm32_projects/STM32H750XBH6/CMake/STM32H750XBH6_Template/.config/FK750M6_XBH6_V0.FLM [flm_region_builder]
0000448 I Loading /home/hao/projects/stm32_projects/STM32H750XBH6/CMake/STM32H750XBH6_Template/build/STM32H750XBH6_Template.bin at 0x90000000 [load_cmd]
[==================================================] 100%
0002723 I Erased 65536 bytes (1 sector), programmed 65536 bytes (8 pages), skipped 0 bytes (0 pages) at 28.14 kB/s [loader]
```

### 实现效果

1. **Bootloader 日志 (运行于片内 Flash)**

   - 程序启动后，首先运行片内 Flash 中的 Bootloader，通过串口打印启动日志。
   - Bootloader 日志截图 （乱码系中文编码问题）

   ![Snipaste_2025-08-03_21-16-45](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20250803215422276.png)

2. **主程序日志 (运行于外部 QSPI Flash)**

   - Bootloader 跳转到 QSPI Flash 中执行主程序，主程序通过 USB 虚拟串口打印运行日志。
   - USB 虚拟串口日志截图

   ![Snipaste_2025-08-03_21-17-15](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20250803215435181.png)

串口波特率115200

### 特别鸣谢

**反客科技**：提供开发板资料和.FLM格式下载算法

**反客交流群的Peakors大佬**：提供STM32H750XHB6的项目模板和openocd外部下载算法，他还制作了很多反客开发板的项目模板和下载算法，[GitHub主页](https://github.com/Peakors)。

**谷歌Gemini**：消化pyocd的文档，提供指导建议和文章排版