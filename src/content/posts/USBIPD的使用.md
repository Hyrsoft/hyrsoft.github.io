---
title: USBIPD的使用
published: 2025-06-01
description: '使用USBIPD把windows上的USB设备挂载进WSL2中'
image: ''
tags: [wsl usbipd]
category: 'tools'
draft: false 
lang: ''
---
### 使用USBIPD将USB设备共享给WSL2  

1. 在windows的powershell中安装**USBIPD**，安装后重启powershell

   ```bash
   winget install usbipd //需要代理
   ```

2. 找到esp32对应的设备，记住BUSID，如：6-4

   ```bash
   usbipd list //列出当前usb设备
   
   6-4    1a86:7522  USB-SERIAL CH340K (COM3)                                      Not shared
   ```

3. 绑定BUSID，运行它被共享到WSL2

   ```bash
   usbipd bind --busid <BUSID>
   # 如：
   usbipd bind --busid 6-4
   
   # 取消共享
   usbipd unbind --busid 6-4
   ```

4. 将USB设备附加到WSL2

   ```bash
   usbipd attach --wsl --busid <BUSID>
   # 如：
   usbipd attach --wsl --busid 6-4
   
   # 取消附加到wsl2上（根据guid）
   usbipd unbind --guid 58e8d17f-29ee-41ba-96b0-10cba8173253
   ```

5. 运行结果：

   ```bash
   # usbipd attach --wsl --busid 6-4
   usbipd: info: Using WSL distribution 'Ubuntu' to attach; the device will be available in all WSL 2 distributions.
   usbipd: info: Using IP address 127.0.0.1 to reach the host.
   ```

6. 在Ubuntu上运行`lsusb`，可以看到多出一个沁恒（ch343）`QinHeng`的设备

   ```bash
   Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
   Bus 001 Device 002: ID 1a86:7522 QinHeng Electronics USB Serial
   Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
   ```

## 图形化软件

::github{repo="nickbeth/wsl-usb-manager"}

