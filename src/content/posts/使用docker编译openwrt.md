---
title: 使用docker编译openwrt
published: 2025-08-02
description: '利用 Docker 环境编译 OpenWrt 固件，以实现编译环境的隔离和一致性'
image: ''
tags: [openwrt, docker]
category: 'linux'
draft: false 
lang: ''
---
介绍如何利用 Docker 环境编译 OpenWrt 固件，以实现编译环境的隔离和一致性。

## 前提条件

| 类别 | 要求 |
| :--- | :--- |
| **网络环境** | 良好的科学上网环境，确保可以顺畅访问 GitHub、OpenWrt 源码等国外资源。 |
| **软件工具** | 在宿主机上预先安装好 `Git` 和 `Docker` (包括 `docker-compose`)。 |
| **个人能力** | 具备基本的 Linux 命令行操作和解决问题的能力。 |

## 环境搭建

### 1\. Dockerfile

创建一个 `Dockerfile` 文件，用于定义编译环境镜像。该镜像基于 Ubuntu 22.04，安装了所有 OpenWrt 官方推荐的编译依赖，并创建了一个名为 `builder` 的非 root 用户以增强安全性。

```dockerfile
# 使用官方推荐的 Ubuntu 22.04 LTS 作为基础镜像
FROM ubuntu:22.04

# 设置环境变量，避免在包安装过程中出现交互式提示
ENV DEBIAN_FRONTEND=noninteractive

# 一次性更新软件源并安装 OpenWrt 编译所需的所有依赖包
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    # 编译工具链
    build-essential \
    clang \
    flex \
    bison \
    g++ \
    gawk \
    gcc-multilib \
    g++-multilib \
    # 文本和版本控制工具
    gettext \
    git \
    subversion \
    # 核心库和开发头文件
    libncurses5-dev \
    libssl-dev \
    zlib1g-dev \
    # Python 相关工具
    python3-distutils \
    python3-docutils \
    python3-pyelftools \
    python3-setuptools \
    # 常用辅助工具
    rsync \
    swig \
    texinfo \
    unzip \
    wget \
    file \
    time \
    # 解决 SSL 证书验证问题的关键包
    ca-certificates \
    # 为非 root 用户提供提权能力的工具
    sudo \
    && \
    # 清理 apt 缓存，减小最终镜像大小
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 创建一个名为 'builder' 的非 root 用户，并赋予其免密 sudo 权限
# 这是为了安全起见，避免在容器中直接使用 root 用户进行编译等操作
RUN useradd -m -s /bin/bash builder && \
    adduser builder sudo && \
    echo 'builder ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# 切换到新创建的非 root 用户
USER builder

# 为 builder 用户设置工作目录
WORKDIR /home/builder

# 设置容器启动时执行的默认命令，即启动一个 bash shell
CMD ["/bin/bash"]
```

### 2\. docker-compose.yml

创建 `docker-compose.yml` 文件来管理容器的构建和运行。该配置会自动构建镜像，并设置一个关键的卷挂载，将宿主机的 `./openwrt` 目录映射到容器内，使得源码和编译产物能够持久化保存在宿主机上。

```yaml
version: '3.8'

services:
  openwrt-builder:
    # 使用当前目录下的 Dockerfile 来构建镜像
    build: .
    # 给构建出的镜像命名
    image: openwrt-builder-image
    # 容器的名字
    container_name: openwrt_build_env
    # 覆盖 Dockerfile 中的 CMD，确保我们能进入 bash
    command: /bin/bash
    # 保持容器在前台运行，并分配一个伪终端 (等同于 -it)
    stdin_open: true
    tty: true
    # 网络模式
    network_mode: host
    # 挂载卷
    volumes:
      # 将宿主机的 ./openwrt 目录挂载到容器的 /home/builder/openwrt
      - ./openwrt:/home/builder/openwrt
    # 设置工作目录，进入容器后会自动切换到这个目录
    working_dir: /home/builder/openwrt
```

### 3\. 推荐目录结构

为了便于管理，建议采用以下目录结构：

```plaintext
~/openwrt_dev/
├── Dockerfile
├── docker-compose.yml
└── openwrt/
```

其中 `openwrt` 目录为 OpenWrt 源码。

## 编译步骤

### 1\. 获取OpenWrt源码

在 `~/openwrt_dev/` 目录下，克隆 OpenWrt 官方源码。

```bash
git clone https://git.openwrt.org/openwrt/openwrt.git
```

### 2\. 构建镜像与启动容器

在 `~/openwrt_dev/` 目录下执行以下命令，Docker Compose 会自动根据 `Dockerfile` 构建镜像，并根据 `docker-compose.yml` 的配置启动容器。

```bash
# 作者安装的docker比较新，把docker-compose整合到docker工具链中了，使用方式和旧版不太一样
docker compose up --build -d

# 旧版 docker-compose
docker-compose up --build -d

# 查看容器情况
docker ps -a

# 启动或关闭我们的容器
docker start openwrt_build_env
docker stop openwrt_build_env
```

使用 `-d` 参数可以在后台启动容器。

### 3\. 进入容器

容器启动后，使用 `docker exec` 命令进入容器内部的 `bash` 环境。

```bash
docker exec -it openwrt_build_env bash
```

### 4\. 执行编译

进入容器后，所有操作都在 `/home/builder/openwrt` 目录下进行。

```bash
# 1. 更新并安装 Feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 2. 生成默认配置文件，并进入配置菜单
#    在这里选择你的目标设备型号、软件包等
#    本文章的重点不在于此，不再展开
make menuconfig

# 3. 下载编译所需的依赖包
make download -j$(nproc)

# 4. 开始编译 (使用与CPU核心数相当的线程数以加快速度)
make -j$(nproc) V=s
```

## 编译产物

编译成功后，生成的固件及相关文件会位于宿主机的 `~/openwrt_dev/openwrt/bin/targets/` 目录下，具体路径取决于你在 `make menuconfig` 中选择的目标平台。例如，`mediatek/filogic` 平台的产物路径如下：

**路径:** `~/openwrt_dev/openwrt/bin/targets/mediatek/filogic/`


![Snipaste_2025-08-02_00-33-20](https://markdownforyuanhao.oss-cn-hangzhou.aliyuncs.com/img1/20250802005325556.png)