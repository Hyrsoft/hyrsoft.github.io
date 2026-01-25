---
title: 在 Wayland (KDE Plasma) 环境下配置 Sunshine 
published: 2026-01-24
description: '在KDE Plasma (Wayland) 以及 NVIDIA 独立显卡环境下配置 Sunshine 服务'
image: ''
tags: [wayland, sunshine]
category: 'linux'
draft: false 
lang: ''
---
在KDE Plasma (Wayland) 以及 NVIDIA 独立显卡环境下配置 Sunshine 服务。

::github{repo="LizardByte/Sunshine"}

## 环境

- **操作系统**: Arch Linux
- **桌面环境**: KDE Plasma (Wayland)
- **显卡**: NVIDIA 独立显卡（需确保已安装 `nvidia` 驱动）

------

## 一、 安装 Sunshine

使用 AUR 助手安装 Sunshine 软件包（其他系统去GitHub仓库release下载对应安装包）：

```bash
# 使用 yay 安装
yay -S sunshine
```

------

## 二、 配置权限与输入环境

为了让 Sunshine 能够捕获输入设备并正常工作，需要配置 `uinput` 权限。

### 1. 创建 udev 规则

创建规则文件以允许 `input` 组访问 `uinput` 设备：

```bash
# 创建规则文件
echo 'KERNEL=="uinput", GROUP="input", MODE="0660", OPTIONS+="static_node=uinput"' | sudo tee /etc/udev/rules.d/60-sunshine.rules

# 确保当前用户在 input 用户组中
sudo usermod -aG input $USER
```

### 2. 设置捕获权限

Sunshine 需要 `cap_sys_admin` 权限来执行特定的屏幕抓取操作：

```bash
sudo setcap cap_sys_admin+ep $(readlink -f $(which sunshine))
```

------

## 三、 检查环境依赖

在 Wayland 环境下，Sunshine 依赖 `xdg-desktop-portal` 来实现屏幕捕获。

| **检查项**      | **命令**                            | **说明**                              |
| --------------- | ----------------------------------- | ------------------------------------- |
| **Portal 插件** | `pacman -Qs xdg-desktop-portal-kde` | 确保安装了针对 KDE 的 Portal 接口实现 |
| **KMS 状态**    | `nvidia-smi`                          | 确保加载了Nvidia显卡驱动 |

------

## 四、 初始启动与配置

1. **首次手动启动**:

   在 KDE 桌面会话内的终端（如 Konsole）中输入 `sunshine` 启动。**注意**：请勿通过远程 SSH 终端执行此操作，否则可能无法正确关联到当前的图形会话。

2. **访问 Web UI**:

   打开浏览器访问 https://localhost:47990。

3. **完成向导**:

   根据页面提示设置用户名、密码并完成基础编码器配置（建议选择 NVENC）。

------

## 五、 配置 Sunshine 为守护进程

使用 **User 模式** 的 Systemd 服务可以确保 Sunshine 在用户登录后自动访问图形和音频服务。

### 1. 服务管理命令

AUR 提供的软件包通常已内置服务文件。

| **操作类型**     | **命令**                           |
| ---------------- | ---------------------------------- |
| **启动服务**     | `systemctl --user start sunshine`  |
| **设置开机自启** | `systemctl --user enable sunshine` |
| **查看实时日志** | `journalctl --user -u sunshine -f` |
| **停止服务**     | `systemctl --user stop sunshine`   |

### 2. 注意事项

- **安全性**: 用户模式服务比系统级服务更安全，且能更好地处理环境变量。
- **日志排查**: 如果无法连接，请使用 `journalctl` 命令检查是否有关于 `vaapi` 或 `cuda` 的报错。

