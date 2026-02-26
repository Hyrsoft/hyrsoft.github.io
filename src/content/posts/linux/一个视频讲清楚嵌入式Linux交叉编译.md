---
title: 视频文案：一个视频讲清楚嵌入式Linux交叉编译
published: 2026-02-19
description: '嵌入式Linux交叉编译的基础概念和技巧'
image: ''
tags: [cross, library, Rust, C++, linux, system]
category: 'linux'
draft: false 
lang: ''
---
<iframe width="100%" height="468" src="//player.bilibili.com/player.html?bvid=BV17XZ9BfEaH&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

（上面视频的文案）

大家新年好！最近突发奇想，决定写一篇关于**嵌入式 Linux 交叉编译**的内容 。

之所以想聊这个，是因为我发现很多刚入门的同学经常在群里问：编译出来的文件怎么烧录到开发板？动态库链接怎么老是报错？ 今天就把我这些年的经验总结分享给大家，希望能帮大家避坑。

------

## 1. 搞清楚谁是 Host，谁是 Target

在聊编译之前，需要先弄明白两个最基本的概念：**Host（主机）** 和 **Target（目标机）** 。



- **Host（编译平台）**：指你干活的机器。比如学单片机时，Host 通常是 **X86 架构的 Windows**；而在嵌入式 Linux 开发中，我们的 Host 一般是 **X86 架构的 Linux 系统**。

- **Target（运行平台）**：指程序最终运行的地方 。

  - **STM32**：属于 ARM Cortex-M 架构（32 位 ARM）。

  - **泰山派 (RK3566)**：四核 Cortex-A55，属于 **ARM64** 或 **aarch64** 架构 。

  - **ESP32**：有 Xtensa 架构，也有 RISC-V 架构 。

    

所以，找编译链的时候，不能只看 Target，还得看 Host 能不能运行这个工具链 。



------

## 2. Target 还要区分 ABI：Bare Metal 和 Linux

打开 ARM 官网的工具链下载页面，你会发现种类繁多 。这里有两个关键点：



### 操作系统区别

- **裸机 (Bare Metal)**：比如编译 STM32 或运行 RTOS 的程序，要选 `arm-none-eabi` 。`EABI` 指的是嵌入式应用二进制接口 。

- **Linux 目标机**：编译像 RK3566 这种跑 Linux 的设备，要选 `aarch64-linux-gnu` 。

  

### 硬件浮点 (Hard Float)

对于 ARMv7 架构（比如三核 A7 的 RK3506），还要注意有没有**硬件浮点支持** 。

- `gnueabihf`：表示支持硬件浮点（Hard Float）。

- `gnueabi`：则没有这个后缀 （软件浮点）。

  

（注：ARMv8/AArch64 之后默认都实现了硬件浮点，所以通常不再专门区分）。

  

 

------

## 3. Linux的核心动态库：Libc 库

在 Linux 上运行的几乎所有用户态程序，都会动态链接 **Libc（C 库）** 。它是对系统调用的封装，是你程序和内核沟通的桥梁 。



1. **Glibc**：GNU Linux 上的标准实现 。

2. **Musl**：一种更轻量、非常适合**静态链接**的实现 。如果你追求程序的兼容性，不想管目标设备上的 Libc 版本是多少，用 Musl 把 C 库直接静态编译进程序里是个好选择 。

   


> **注意**：做单片机开发的同学可能没这个概念，因为单片机程序是全静态编译后直接烧录闪存的，没有操作系统的动态链接概念 。

------



## 4. 交叉编译动态链接：Sysroot

这是最让新人头疼的地方。你的程序不只链接 C 库，还可能链接音频库（如 ALSA）、视频编解码库等 。 这些库都在开发板上（`/usr/lib` 目录下），你的主机（Host）里没有，编译时去哪里找？

这时候就需要 **Sysroot** 。

**Sysroot 的作用**：它相当于在你的主机上“伪造”了一个目标机的根目录 。它同步了目标机的 `/lib`、`/usr/include`、`/usr/lib` 等目录 。



### 如何获取 Sysroot？

- **Buildroot 方案**：如果你用 Buildroot SDK，它编译完后会在 `output/target` 下生成一个备份，这就是现成的 Sysroot 。

  > 任何由自己编译的 rootfs，都是相同的原理

- **主流发行版（Ubuntu/Debian）**：它们通过 `apt` 管理软件，主机里没有现成的备份 。

  > **技巧**：你可以用 `rsync` 工具，把开发板上的 `/usr/lib` 和 `/usr/include` 增量同步到主机的一个目录里，然后编译时指定这个目录为 Sysroot 。

  

  

------

## 5. 总结

无论是 C/C++ 还是 Rust，交叉编译的核心逻辑是一样的 ：

1. 设置好交叉工具链（GCC/LLVM）的路径 。

2. **指定正确的 Sysroot 路径**，让编译器能找到目标平台的头文件和动态库 。