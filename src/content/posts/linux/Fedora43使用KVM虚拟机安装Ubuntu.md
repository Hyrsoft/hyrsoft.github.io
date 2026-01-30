---
title: Fedora43使用KVM虚拟机安装Ubuntu
published: 2025-10-30
description: '基于Fedora的KVM虚拟机安装Ubuntu作为开发环境'
image: ''
tags: [Virtual-Machine]
category: 'linux'
draft: false 
lang: ''
---

# Fedora 43 KVM 虚拟机安装 Ubuntu 记录

本文档记录在 Fedora 43 宿主机上使用 KVM 部署 Ubuntu (22.04/18.04) 虚拟机的完整流程，涵盖环境搭建、虚拟机创建、资源共享、网络桥接及快照管理。

## 1. 环境准备与虚拟化支持

首先需要在宿主机开启虚拟化支持并安装必要的管理工具。

### 1.1 安装 KVM 相关组件

```bash
# 安装虚拟化组包
sudo dnf groupinstall "Virtualization"

# 安装安装工具与查看器
sudo dnf install virt-install virt-viewer
```

### 1.2 启动服务

```bash
sudo systemctl enable --now libvirtd
```

------

## 2. 构建虚拟机

### 2.1 准备镜像与目录

建议将 ISO 镜像移动到 `libvirt` 默认目录以避免权限问题。

```bash
# 创建虚拟机镜像存放目录 (可选，默认在 /var/lib/libvirt/images/)
sudo mkdir -p /var/lib/libvirt/images

# 移动 ISO 镜像
sudo mv /home/hao/ubuntu-22.04.5-desktop-amd64.iso /var/lib/libvirt/images/
```

### 2.2 执行安装命令

使用 `virt-install` 创建名为 `vivado-lab` 的虚拟机，8GB内存，8个处理器逻辑核心，500GB存储空间。按需修改，核心数和内存都可以后期调整

```bash
# 预先分配足够的磁盘空间
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/vivado-lab.qcow2 500G

# 确保所有者正确 (libvirt/qemu 进程通常运行在 qemu 用户下)
sudo chown qemu:qemu /var/lib/libvirt/images/vivado-lab.qcow2

# 恢复默认上下文
sudo restorecon -Rv /var/lib/libvirt/images/

# 再次确认关闭 SELinux (临时测试用，排除干扰)
sudo setenforce 0

sudo virt-install \
  --name vivado-lab \
  --memory 8192 \
  --vcpus 8 \
  --cpu host-model \
  --disk path=/var/lib/libvirt/images/vivado-lab.qcow2,bus=virtio,format=qcow2 \
  --cdrom /var/lib/libvirt/images/ubuntu-22.04.5-desktop-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default,model=virtio \
  --graphics spice,listen=none,gl.enable=no \
  --video virtio \
  --channel spicevmc \
  --check disk_size=off \
  --noautoconsole
```

**参数说明：**

| **参数**                 | **说明**                               |
| ------------------------ | -------------------------------------- |
| `--cpu host-passthrough` | 透传宿主机 CPU 特性，性能最佳          |
| `--graphics spice`       | 使用 SPICE 图形协议                    |
| `--disk ...`             | 指定磁盘路径、大小 、总线类型 (virtio) |
| `--noautoconsole`        | 自动后台安装，不直接弹出控制台         |

### 2.3 访问与管理

```bash
# 通过 virt-viewer 进入虚拟机界面
sudo virt-viewer --attach vivado-lab

# 打开图形化管理器
sudo virt-manager
```

**常用管理命令：**

```bash
# 关闭虚拟机
sudo virsh shutdown vivado-lab

# 开启虚拟机
sudo virsh start vivado-lab

# 查看所有虚拟机状态
sudo virsh list --all

# 删除虚拟机
sudo virsh undefine vivado-lab --remove-all-storage --delete-snapshots --nvram
```

------

## 3. 配置宿主机与虚拟机共享目录 (VirtioFS)

将宿主机的 `/var/lib/libvirt/share` 目录挂载到虚拟机的 `/mnt/share`。

### 3.1 宿主机目录权限配置

必须正确设置目录的所有者和 SELinux 上下文，否则虚拟机无法访问。

```bash
# 1. 创建目录
sudo mkdir -p /var/lib/libvirt/share

# 2. 设置归属 (移交给 qemu 用户)
sudo chown qemu:qemu /var/lib/libvirt/share

# 3. 设置 SELinux 上下文 (放行 virt 访问)
sudo chcon -t virt_image_t /var/lib/libvirt/share
```

### 3.2 编辑虚拟机 XML 配置

使用 `sudo virsh edit vivado-lab` 修改配置文件。

步骤 1：添加 Filesystem 设备

在 <devices> 段落下添加：

```xml
<filesystem type='mount' accessmode='passthrough'>
  <driver type='virtiofs'/>
  <source dir='/var/lib/libvirt/share'/>
  <target dir='host_share'/>
</filesystem>
```

步骤 2：开启共享内存

在 <domain> 段落下（通常在 CPU 配置附近）添加或修改：

```xml
<domain type='kvm'>
  ...
  <memoryBacking>
    <source type='memfd'/>
    <access mode='shared'/>
  </memoryBacking>
  ...
</domain>
```

### 3.3 虚拟机内部挂载

启动虚拟机后，在内部执行：

```bash
sudo mkdir /mnt/host

# 挂载共享目录 (target tag 为 host_share)
sudo mount -t virtiofs host_share /mnt/host
```

------

## 4. 网络配置 (桥接模式)

将宿主机物理网卡桥接到 `br0`，并让虚拟机连接该网桥，使其获得独立局域网 IP。

### 4.1 宿主机创建网桥

物理有线网卡名称为 `enp6s0f4u1u4`。

```bash
# 1. 创建网桥从属连接
sudo nmcli con add type bridge-slave ifname enp6s0f4u1u4 master br0

# 2. 删除旧的有线连接 (防止冲突)
sudo nmcli con delete "有线连接 1"

# 3. 激活网桥
sudo nmcli con up br0
```

### 4.2 虚拟机网络设置

在 `virt-manager` 中将虚拟机的网络接口修改为 **桥接设备 -> br0**，或编辑 XML 将 interface type 改为 `bridge`。

![截屏2025-11-21 21.02.09](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20251121210214247.png)

------

## 5. 虚拟机基础服务配置 

### 5.1 配置 APT 镜像源 (清华源)

加速软件安装与更新。

```bash
sudo sed -i.bak 's/cn.archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list && \
sudo sed -i 's/security.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list && \
sudo apt update
```

### 5.2 安装 SSH 服务

```bash
sudo apt install openssh-server

# 启动并设置开机自启
sudo systemctl enable --now ssh

# 检查状态
sudo systemctl status ssh
```

------

## 6. Tailscale 子网路由配置 (宿主机)

配置宿主机作为 Tailscale 子网路由器，转发 `192.168.137.0/24` 网段流量。

### 6.1 开启内核转发

编辑 `/etc/sysctl.d/99-tailscale.conf`：

```toml
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

应用配置：

```bash
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 6.2 广播路由

```bash
# 启动 Tailscale 并广播子网
sudo tailscale up --advertise-routes=192.168.137.0/24
```

验证：外部节点 Ping 虚拟机 IP (e.g., `192.168.137.226`) 应正常连通。

![截屏2025-11-21 21.23.12](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20251121212315323.png)

------

## 7. 虚拟机快照管理

使用 `virsh snapshot` 系列命令管理系统状态。

### 7.1 创建快照

创建完整系统快照 (Live Snapshot)

保留当前运行状态 (内存+磁盘)。

```bash
sudo virsh snapshot-create-as \
  --domain vivado-lab \
  --name "snap-20251122-base" \
  --description "Base install with network and ssh configured"
```

创建关机状态快照 (Disk-only)

占用空间小，适合纯磁盘备份。

```bash
# 先关闭虚拟机
sudo virsh shutdown vivado-lab

# 创建快照
sudo virsh snapshot-create-as \
  --domain vivado-lab \
  --name "env-ready-2025" \
  --description "Setup with SSH, and Tsinghua Mirror" \
  --disk-only \
  --atomic
```

### 7.2 查看与管理快照

```bash
# 列出所有快照
sudo virsh snapshot-list --domain vivado-lab

# 查看快照详细信息
sudo virsh snapshot-info --domain vivado-lab --snapshotname "env-ready-2025"

# 恢复到指定快照
sudo virsh snapshot-revert --domain vivado-lab --snapshotname "env-ready-2025"
# 恢复并暂停 (便于检查)
# sudo virsh snapshot-revert --domain vivado-lab --snapshotname "env-ready-2025" --paused

# 删除快照
sudo virsh snapshot-delete --domain vivado-lab --snapshotname "snap-20251122-base"
```
