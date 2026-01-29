---
title: 使用GDB调试嵌入式设备
published: 2026-01-30
description: '介绍在嵌入式 Linux 设备中使用 GDB 进行程序调试的方法'
image: ''
tags: [GDB, linux, C, C++, Debug]
category: 'tools'
draft: false 
lang: ''
---
本文主要介绍在嵌入式 Linux 设备中使用 GDB 进行程序调试的方法。对于单片机开发，虽然可以使用 IDE 集成的调试功能，或通过 `pyocd`、`openocd` 配合 `DAPLink` 等硬件调试探针，但其核心原理与 GDB 调试是一致的。

------

## 环境准备

在进行调试前，需确保开发环境（如 Buildroot）已安装必要的调试组件。

### 1. 安装组件

通过 `menuconfig` 在根文件系统中启用以下工具：

- **gdb**: 运行在开发板上的完整调试器。
- **gdbserver**: 运行在开发板上的轻量化调试代理（推荐用于交叉调试）。

验证安装：

```bash
[root@luckfox root]# gdb --version
GNU gdb (GDB) 10.2
[root@luckfox root]# gdbserver --version
GNU gdbserver (GDB) 10.2
```

### 2. 编译配置

为了生成调试符号表并确保代码执行顺序与源码一致，必须在编译时开启 `-g` 参数并尽量禁用优化 `-O0`。

```cmake
# 例如使用CMake进行构建时，设置参数
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -g")
```

------

## 常用调试途径对比

| **途径**       | **实现方式**                                   | **适用场景**                                     |
| -------------- | ---------------------------------------------- | ------------------------------------------------ |
| **GDB 命令行** | 直接在板端通过 GDB 运行程序或分析 Core Dump    | 快速定位崩溃点、打印堆栈、资源受限无法连接主机时 |
| **单步调试**   | 通过 `gdbserver` 连接主机（如 VSCode）远程调试 | 逻辑分析、对照源码逐行排查、复杂业务流程调试     |

------

## 一、 GDB 命令行调试

### 1. 场景一：实时捕获（Live Debug）

针对可以稳定复现的程序崩溃（Segmentation fault）。

- **启动调试**：

  ```bash
  gdb ./your_app
  ```

- **常用交互命令**：

| **命令**                        | **说明**                                       |
| ------------------------------- | ---------------------------------------------- |
| `set args --config ./test.conf` | 设置程序启动参数                               |
| `run` (r)                       | 开始运行程序                                   |
| `bt` (backtrace)                | 打印当前简略调用堆栈                           |
| `bt full`                       | 打印堆栈及每一层函数的局部变量值               |
| `thread apply all bt`           | 查看所有线程的堆栈（用于分析多线程死锁或崩溃） |

### 2. 场景二：Core Dump 核心转储

用于处理偶发性崩溃，通过事后“尸检”定位问题。

- **启用生成**：

  ```bash
  ulimit -c unlimited  # 设置 core 文件大小不受限制
  ```

- **设置路径**（可选）：

  ```bash
  # 将 core 文件统一存放到 /tmp，并按进程名、PID 和时间命名
  echo "/tmp/core-%e-%p-%t" > /proc/sys/kernel/core_pattern
  ```

- **加载分析**：

  ```bash
  # 必须同时提供可执行文件和 core 文件
  gdb ./your_app core
  ```

### 3. 场景三：深入崩溃点

| **操作**     | **命令示例**                | **功能描述**                                 |
| ------------ | --------------------------- | -------------------------------------------- |
| **切换帧**   | `frame 2`                   | 切换到堆栈中的指定层级（如切换回应用代码层） |
| **查看源码** | `list`                      | 显示当前断点或崩溃点附近的源码               |
| **检查变量** | `print *my_ptr`             | 打印变量值或解引用指针                       |
| **检查状态** | `info locals` / `info args` | 打印当前函数的所有局部变量或输入参数         |
| **汇编调试** | `disassemble`               | 在无源码时查看当前函数的汇编代码             |

------

## 二、 远程单步调试

在嵌入式开发中，通常在板端运行 `gdbserver`（节省空间），在 Host 端（PC）使用带符号表的程序和源码进行可视化调试。

### 1. 板端操作

启动 `gdbserver` 并监听调试端口（如 `1234`）：

```bash
gdbserver :1234 ./my_app
```

### 2. Host 端配置 (VSCode)

需安装 **C/C++ (cpptools)** 扩展。创建 `.vscode/launch.json`：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Luckfox Remote Debug",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/my_app", // 必须是带调试符号的可执行文件
            "args": [],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/opt/luckfox-sdk/tools/linux/toolchain/.../arm-linux-gnueabihf-gdb", // 交叉编译器的 GDB 路径
            "miDebuggerServerAddress": "192.168.1.123:1234", // 板端 IP 和端口
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "Set Sysroot for Cross-Debugging",
                    "text": "set sysroot ${workspaceFolder}/buildroot/output/staging", // 关键：指定交叉编译系统库路径
                    "ignoreFailures": false
                }
            ]
        }
    ]
}
```

配置完成后，在 VSCode 中按 `F5` 即可进入图形化单步调试界面。