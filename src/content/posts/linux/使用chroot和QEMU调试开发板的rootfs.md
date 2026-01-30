---
title: 使用chroot和qemu调试开发板的rootfs
published: 2025-07-28
description: '一般用于系统移植或者调试'
image: ''
tags: [qemu, rootfs, chroot]
category: 'linux'
draft: false 
lang: ''
---
## 一、`chroot` 与 qemu 简介

### `chroot` (Change Root)

`chroot` 是 Linux 系统中的一个命令及系统调用，用于将一个进程及其子进程的根目录切换到文件系统中的一个新位置。这使得该进程无法访问或命名新根目录之外的文件。

**主要用途：**

  * **系统修复**：当系统无法启动时，可以使用 Live CD 启动，然后 `chroot` 到损坏的系统中进行修复。
  * **安全隔离**：限制特定进程的文件系统访问权限，创建一个简易的沙箱环境。
  * **交叉编译**：在 x86 主机上为 ARM 等不同架构的设备构建和测试软件。
  * **环境测试**：在当前系统中模拟另一个 Linux 发行版环境，例如在 Ubuntu 中运行 Debian 环境。

### `qemu-user-static`

当在 x86 架构的主机上 `chroot` 到一个为 ARM 等异构平台构建的根文件系统时，会因为 CPU 指令集不兼容而导致无法执行任何程序。`qemu-user-static` 通过 `binfmt_misc` 机制解决了这个问题，它能够透明地解释和执行异构架构的二进制文件，从而在主机上模拟目标硬件环境。

## 二、操作流程

以用**Ubuntu**系统调试立创泰山派的**Ubuntu22.04 rootfs**为例

### 1\. 安装 `qemu-user-static`

首先，在主机系统中安装 `qemu-user-static` 软件包。

```bash
sudo apt update
sudo apt install qemu-user-static
```

安装完成后，将对应目标架构的 QEMU 静态二进制文件复制到开发板根文件系统的 `usr/bin` 目录下。例如，对于 ARM 架构：

```bash
# 假设 "ubuntu/" 是开发板根文件系统的挂载点
sudo cp /usr/bin/qemu-arm-static ubuntu/usr/bin/
sudo cp /usr/bin/qemu-aarch64-static ubuntu/usr/bin/
```

为了确保 QEMU 能够自动处理异构架构的二进制文件，需要检查 `binfmt_misc` 的配置是否已启用。

```bash
ls -l /proc/sys/fs/binfmt_misc/
# 检查对应的 qemu-aarch64 解释器是否启用
cat /proc/sys/fs/binfmt_misc/qemu-aarch64
```

理想情况下，`cat` 命令的输出应显示 `enabled`。


默认情况下qemu会自动注册，但是笔者使用的wsl2可能有bug，不会自动注册，这时候需要手动注册：


```bash
# rsicv64
sudo sh -c 'echo ":qemu-riscv64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xf3\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-riscv64-static:F" > /proc/sys/fs/binfmt_misc/register'

# aarch64
sudo sh -c 'echo ":qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64-static:F" > /proc/sys/fs/binfmt_misc/register'
```

### 2\. 挂载根文件系统 (rootfs)

为了方便地挂载和卸载 `chroot` 环境所需的虚拟文件系统，可以创建一个脚本。

**创建 `mount.sh` 脚本：**

```bash
#!/bin/bash
# mount.sh - Mounts/unmounts essential filesystems for chroot

function mnt() {
    echo "MOUNTING..."
    sudo mount -t proc /proc ${2}proc
    sudo mount -t sysfs /sys ${2}sys
    sudo mount -o bind /dev ${2}dev
    sudo mount -o bind /dev/pts ${2}dev/pts
    echo "Entering chroot environment..."
    sudo chroot ${2}
}

function umnt() {
    echo "UNMOUNTING..."
    sudo umount ${2}proc
    sudo umount ${2}sys
    sudo umount ${2}dev/pts
    sudo umount ${2}dev
}

if [ "$1" == "-m" ] && [ -n "$2" ]; then
    mnt $1 $2
elif [ "$1" == "-u" ] && [ -n "$2" ]; then
    umnt $1 $2
else
    echo "Usage: $0 [-m|-u] <rootfs_path>"
    echo "  -m: Mount and chroot into the rootfs"
    echo "  -u: Unmount the rootfs"
    echo "Example: $0 -m /path/to/rootfs/"
fi
```

**赋予脚本执行权限：**

```bash
chmod +x mount.sh
```

也可以使用这个健壮性更好的python脚本

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import sys
import subprocess
import shutil

# --- 用户配置 ---
# 在这里修改为要挂载的根文件系统的绝对路径
ROOTFS_PATH = "/home/hao/k230_sdk/output/k230_canmv_lckfb_defconfig/images/debian13"

# 如果需要跨架构 chroot (例如在 x86 电脑上 chroot ARM 系统),
# 指定 QEMU 静态二进制文件的路径。如果不需要，请留空 ""。
# 通常在 /usr/bin/ 目录下, 例如 qemu-aarch64-static, qemu-arm-static 等。
QEMU_STATIC_BINARY = "/usr/bin/qemu-riscv64-static"
# --- 配置结束 ---


def run_command(command, check=True):
    """执行一个 shell 命令并处理可能发生的错误"""
    print(f"🚀 执行: {' '.join(command)}")
    try:
        subprocess.run(command, check=check)
    except FileNotFoundError:
        print(f"❌ 命令未找到: {command[0]}。请确保该程序已安装并在 PATH 环境变量中。", file=sys.stderr)
        sys.exit(1)
    except subprocess.CalledProcessError as e:
        print(f"❌ 命令执行失败: {' '.join(e.cmd)} (返回码: {e.returncode})", file=sys.stderr)
        if not check:
             print("   (警告: 此命令失败，但程序将继续执行)")
        else:
            sys.exit(1)


def unmount_filesystems():
    """
    按正确顺序卸载所有 chroot 文件系统。
    该函数设计得非常健壮，会检查每个挂载点是否存在且已挂载。
    """
    print("\n🧹 开始安全卸载程序...")
    mount_points = ['dev/pts', 'dev', 'sys', 'proc']

    for mp in mount_points:
        target_path = os.path.join(ROOTFS_PATH, mp)
        if os.path.exists(target_path) and os.path.ismount(target_path):
            run_command(['sudo', 'umount', '-l', target_path], check=False) # 使用 -l (lazy) 更安全

    if QEMU_STATIC_BINARY:
        qemu_dest_path = os.path.join(ROOTFS_PATH, 'usr', 'bin', os.path.basename(QEMU_STATIC_BINARY))
        if os.path.exists(qemu_dest_path):
            print(f"   清理 QEMU 模拟器: {qemu_dest_path}")
            run_command(['sudo', 'rm', qemu_dest_path])

    print("✅ 清理完成。")


def mount_and_chroot():
    """
    挂载所需的文件系统并进入 chroot 环境。
    使用 try...finally 确保无论 chroot 内部发生什么，都会执行卸载。
    """
    try:
        print("🛠️  开始挂载 chroot 所需的文件系统...")
        mounts = [
            ('proc', 'proc', 'proc', None),
            ('sysfs', 'sys', 'sysfs', None),
            ('/dev', 'dev', None, 'bind'),
            ('/dev/pts', 'dev/pts', None, 'bind'),
        ]

        for source, dest_subdir, fstype, options in mounts:
            target_path = os.path.join(ROOTFS_PATH, dest_subdir)
            if not os.path.exists(target_path):
                print(f"   创建缺失的目录: {target_path}")
                run_command(['sudo', 'mkdir', '-p', target_path])

            command = ['sudo', 'mount']
            if fstype: command.extend(['-t', fstype])
            if options == 'bind': command.extend(['-o', 'bind'])
            command.extend([source, target_path])
            run_command(command)

        if QEMU_STATIC_BINARY:
            if not os.path.exists(QEMU_STATIC_BINARY):
                print(f"❌ 错误: QEMU 模拟器 '{QEMU_STATIC_BINARY}' 未在您的主机上找到。", file=sys.stderr)
                print("   如果您正在进行跨架构 chroot，请安装它 (例如: 'sudo apt install qemu-user-static')。", file=sys.stderr)
                return

            ### --- 新增功能：检查 binfmt_misc 是否启用 --- ###
            # 从 QEMU 二进制文件名推断出 binfmt_misc 的处理器名
            # 例如 "qemu-aarch64-static" -> "qemu-aarch64"
            handler_name = os.path.basename(QEMU_STATIC_BINARY).removesuffix('-static')
            binfmt_path = f"/proc/sys/fs/binfmt_misc/{handler_name}"
            
            print(f"   检查 QEMU binfmt_misc 处理器: {binfmt_path}")
            if not os.path.exists(binfmt_path):
                print(f"❌ 错误: QEMU binfmt_misc 处理器未启用 ({binfmt_path} 不存在)。", file=sys.stderr)
                print("   这意味着您的主机内核可能无法直接运行目标架构的二进制文件。", file=sys.stderr)
                print("   请尝试通过重启服务来注册处理器，例如运行:", file=sys.stderr)
                print("     sudo systemctl restart systemd-binfmt.service", file=sys.stderr)
                print("   或者:", file=sys.stderr)
                print("     sudo update-binfmts --enable", file=sys.stderr)
                return # 提前中止，finally 块会负责清理
            
            print("   ✅ QEMU binfmt_misc 处理器已启用。")
            ### --- 新增功能结束 --- ###

            qemu_dest_path = os.path.join(ROOTFS_PATH, 'usr', 'bin', os.path.basename(QEMU_STATIC_BINARY))
            print(f"   复制 QEMU 模拟器到 chroot 环境: {qemu_dest_path}")
            run_command(['sudo', 'cp', QEMU_STATIC_BINARY, qemu_dest_path])

        print("\n✅ 挂载完成。即将进入 chroot 环境...")
        print("   在 chroot 环境中，您可以执行所需命令。")
        print("   完成后，请键入 'exit' 以退出 chroot 并自动卸载所有文件系统。")
        
        run_command(['sudo', 'chroot', ROOTFS_PATH])

    except Exception as e:
        print(f"\n❌ 在挂载或 chroot 过程中发生意外错误: {e}", file=sys.stderr)
    finally:
        print("\n🚪 已退出 chroot 环境或发生错误。")
        unmount_filesystems()


def main():
    """脚本主入口"""
    if os.geteuid() != 0:
        print("❌ 错误: 此脚本需要 root 权限。请使用 'sudo' 运行。", file=sys.stderr)
        sys.exit(1)

    if not os.path.isdir(ROOTFS_PATH):
        print(f"❌ 错误: 配置的根文件系统路径 '{ROOTFS_PATH}' 不是一个有效的目录。", file=sys.stderr)
        sys.exit(1)

    if len(sys.argv) != 2 or sys.argv[1] not in ['-m', '-u']:
        print(f"用法: sudo {sys.argv[0]} [-m|-u]")
        print("  -m: 挂载文件系统并进入 chroot 环境")
        print("  -u: 仅卸载文件系统 (用于手动清理)")
        sys.exit(1)

    action = sys.argv[1]
    if action == '-m':
        mount_and_chroot()
    elif action == '-u':
        unmount_filesystems()

if __name__ == '__main__':
    main()
```

使用方式

1. **配置**：打开 `manage_chroot.py` 文件，修改顶部的 `ROOTFS_PATH` 变量，使其指向你的根文件系统目录（例如 `/home/user/my_project/rootfs`）。如果需要，同时配置 `QEMU_STATIC_BINARY`。

2. **挂载并进入 Chroot**： 在终端中运行以下命令。脚本将挂载所有必要的文件系统，然后自动带你进入 `chroot` 环境。

   ```bash
   sudo python3 manage_chroot.py -m
   ```

   当你完成在 `chroot` 环境中的工作后，只需输入 `exit` 并按回车。脚本会自动检测到退出，并执行所有卸载和清理操作。

3. **仅执行卸载**： 如果在某些异常情况下，你需要手动执行清理，可以运行以下命令。它会安全地卸载所有相关的挂载点。

   ```bash
   sudo python3 manage_chroot.py -u
   ```

### 3\. 进入 `chroot` 环境并进行系统配置

使用上一步创建的脚本挂载并进入开发板的根文件系统。

```bash
# 假设开发板根文件系统位于当前目录下的 "ubuntu/"
sudo ./mount.sh -m ubuntu/
```

成功进入后，将处于开发板的模拟环境中，可以执行系统配置任务。

**以下是曾经给泰山派移植ubuntu22.04时的记录，仅供参考**

泰山派sdk可用：

::github{repo="CmST0us/tspi-linux-sdk"}


#### a. 更新软件源及安装必要软件

```bash
# 某些系统可能需要赋予/tmp目录写权限
chmod 777 /tmp

apt update
apt install -y sudo vim udev net-tools ethtool udhcpc netplan.io \
               language-pack-en-base iputils-ping openssh-sftp-server \
               ntp usbutils alsa-utils libmtp9 language-pack-zh-han* \
               bluetooth* bluez* blueman* wireless-tools network-manager dialog

# 清理不需要的软件包
apt-get remove --purge libreoffice* -y
apt-get autoremove -y
```

#### b. 添加用户及配置权限

```bash
# 1. 添加一个新用户（例如 tspi）
adduser tspi

# 2. 将新用户添加到 sudo 组以赋予管理员权限
adduser tspi sudo

# 3. 修改 root 用户的密码
passwd root
```

#### c. 优化开机网络等待时间

默认情况下，系统启动时可能会等待网络配置长达5分钟。通过修改 systemd 服务，可以缩短或禁用此等待时间。

**编辑服务文件：**

```bash
vim /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service
```

**修改 `ExecStart` 行：**
将 `timeout` 参数改为一个较小的值（如 5 秒）或 `0`（禁用）。

```ini
[Service]
ExecStart=/usr/bin/nm-online -s -q --timeout=5
```

#### d. 添加分区自动扩容服务（重要）

为了让烧录到 SD 卡或 eMMC 的文件系统在首次启动时自动扩展以使用整个分区空间，可以创建一个系统服务。

**(1) 创建 `resize2fs.sh` 脚本：**

```bash
vim /etc/init.d/resize2fs.sh
```

**添加以下内容：**
该脚本会检查一个标志文件，如果不存在，则执行分区扩容并创建该标志文件，确保扩容只执行一次。

```bash
#!/bin/bash -e
# Resize the root filesystem partition, e.g., /dev/mmcblk0p6

PARTITION_TO_RESIZE="/dev/mmcblk0p6"
FLAG_FILE="/usr/local/boot_flag"

if [ ! -e "$FLAG_FILE" ] ; then
  echo "Resizing $PARTITION_TO_RESIZE..."
  resize2fs $PARTITION_TO_RESIZE
  touch "$FLAG_FILE"
  echo "Resize complete."
fi
```

**(2) 赋予脚本执行权限：**

```bash
chmod +x /etc/init.d/resize2fs.sh
```

**(3) 创建 systemd 服务来执行此脚本：**

```bash
vim /lib/systemd/system/resize2fs.service
```

**添加服务定义：**

```ini
[Unit]
Description=Resize root filesystem on first boot
DefaultDependencies=no
After=systemd-remount-fs.service
Before=local-fs.target

[Service]
Type=oneshot
ExecStart=/etc/init.d/resize2fs.sh
StandardOutput=journal
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**(4) 启用该服务：**

```bash
systemctl enable resize2fs.service
```

### 4\. 退出并卸载根文件系统

完成所有配置后，退出 `chroot` 环境。

```bash
exit
```

返回到主机系统后，使用脚本卸载之前挂载的虚拟文件系统（如果用上面提供的python脚本，当退出时会自动卸载，不用手动操作）。

```bash
sudo ./mount.sh -u ubuntu/
```