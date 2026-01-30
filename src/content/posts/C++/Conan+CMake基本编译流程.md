---
title: Conan+CMake基本编译流程
published: 2025-10-21
description: '在CMake构建的项目中使用Conan2作为包管理器'
image: ''
tags: [CMake, Conan]
category: 'C++'
draft: false 
lang: ''
---
# Conan+CMake基本编译流程

## 本地编译

### 1\. Conan 安装依赖

首先，使用 Conan 安装项目所需的第三方包。Conan 会根据构建类型（Debug/Release）将生成的文件放置在不同的目录中。

  * 默认情况下，Conan 会创建 `build/Release/generators` 目录来存放生成文件。

    ```bash
    conan install . --build=missing
    ```

  * 可以明确指定构建类型，例如 `Debug`。

    ```bash
    conan install . --build=missing -s build_type=Debug
    ```

  * `--output-folder` 参数可以指定输出目录，但通常不是必需的，因为 Conan 会根据配置自动管理路径。

### 2\. CMake 配置与编译

Conan 在安装依赖后，会自动生成一个 CMake 预设文件 (`CMakeUserPresets.json`)，其中包含了配置和编译所需的全部信息。因此，推荐直接使用预设来简化操作。

#### 使用 CMake 预设（推荐）

  * **配置 CMake**

    使用 `conan-debug` 或 `conan-release` 预设来配置项目。

    ```bash
    # 配置 Debug 版本
    cmake --preset conan-debug
    
    # 配置 Release 版本
    cmake --preset conan-release
    ```

  * **编译**

    同样使用预设来执行编译。

    ```bash
    # 编译 Debug 版本
    cmake --build --preset conan-debug
    
    # 编译 Release 版本
    cmake --build --preset conan-release
    ```

#### 手动配置

如果不使用预设，则需要手动指定工具链文件和构建类型。

  * **配置 CMake**

    ```bash
    # 配置 Release 版本
    cmake -S . -B build/Release -G Ninja \
          -DCMAKE_TOOLCHAIN_FILE=build/Release/generators/conan_toolchain.cmake \
          -DCMAKE_BUILD_TYPE=Release
          
    # 配置 Debug 版本
    cmake -S . -B build/Debug -G Ninja \
          -DCMAKE_TOOLCHAIN_FILE=build/Debug/generators/conan_toolchain.cmake \
          -DCMAKE_BUILD_TYPE=Debug
    ```

  * **编译**

    ```bash
    # 编译 Release 版本
    cmake --build build/Release
    
    # 编译 Debug 版本
    cmake --build build/Debug
    ```

## 交叉编译

### 1\. 创建 Profile 文件

首先，需要为目标平台创建一个 Conan Profile 文件。例如，为 aarch64 架构创建一个 `profiles/aarch64_profile` 文件。

```toml
[settings]
os=Linux
arch=armv8
compiler=gcc
compiler.version=15.2
compiler.libcxx=libstdc++11
compiler.cppstd=20

[conf]
# 告诉 Conan 加载我们自定义的 CMake 工具链文件
# 使用 tools.cmake.cmaketoolchain:user_toolchain 来指定
# 注意: ${profile_dir} 是 Conan 变量，但实践中建议使用绝对路径以确保正确识别
tools.cmake.cmaketoolchain:user_toolchain=["/path/to/your/project/cmake/aarch64-toolchain.cmake"]
```

在对应的 `.cmake` 工具链文件中，你需要设置交叉编译所需的编译器、系统根目录等环境变量。

### 2\. 执行编译流程

交叉编译的流程与本地编译非常相似，区别在于需要在 `conan install` 命令中通过 `-pr:h` 参数指定 Host 平台的 Profile 文件。

  * **Conan 安装依赖**

    ```bash
    conan install . --build=missing -s build_type=Debug -pr:h profiles/aarch64_profile
    ```

  * **配置 CMake**

    Conan 同样会生成适用于交叉编译的 CMake 预设。

    ```bash
    cmake --preset conan-debug
    cmake --preset conan-release
    ```

  * **编译**

    ```bash
    cmake --build --preset conan-debug
    cmake --build --preset conan-release
    ```

