---
title: C++学习笔记：std::unique_ptr自定义删除器
published: 2025-12-15
description: '使用std::unique_ptr，对c-style的资源进行RAII式的安全封装'
image: ''
tags: [C++]
category: 'C++'
draft: false 
lang: ''
---
## 一、直接在`std::unique_ptr`中调用自定义删除器

### 代码：

```cpp
#include <iostream>
#include <memory>

// 这里有一个c-style的资源，清理它需要手动
// 这个XPacket是外部提供的，不好进行重构（添加构造和析构函数）
struct XPacket {
    unsigned char* data = nullptr; //一个需要手动释放的堆内存指针
    int size = 0;
};

// 专门创建一个PacketDelete类来进行XPacket的清理
class PacketDelete {
public:
    // 重载了()
    void operator() (XPacket* p) const {
        std::cout << "call PacketDelete()" << std::endl;
        // 在这里释放XPacket的资源，或者调用资源清理的接口
        std::free(p->data);
        delete p;
    }
};

int main() {
    // 自定义空间删除方法，希望智能指针离开作用域后调用指定的资源清理方法
  	// 而不是单纯的结束XPacket的生命周期（它是一个c-style结构体，没有专门的析构函数进行资源清理）
    std::unique_ptr<XPacket, PacketDelete> p1 (new XPacket);

   
    // std::unique_ptr<XPacket, PacketDelete> p3 (new XPacket);
    // p3.get_deleter()(p2.get());
  
  	// std::make_unique()不支持自定义删除器，只能用std::unique_ptr<>来创建
    // auto p2 = std::make_unique<XPacket, PacketDelete>();
    return 0;
}
```

### 解析

### 1、`PacketDelete`类的函数调用运算符

```cpp
class PacketDelete {
public:
    // 重载了()
    void operator() (XPacket* p) const {
        std::cout << "call PacketDelete()" << std::endl;
        delete p->data;
        delete p;
    }
};
```

这是一个**函数调用运算符 (Function Call Operator)**，俗称 **仿函数 (Functor)**。它使得 `PacketDelete` 类的对象可以像函数一样被调用。

### 2、`std::unique_ptr`调用自定义删除器

```cpp
std::unique_ptr<XPacket, PacketDelete> p1 (new XPacket);
```

当 `p1` 智能指针离开作用域时，它会执行以下步骤：

1. 获取它持有的 `XPacket*` 裸指针。
2. 获取它的删除器对象（类型为 `PacketDelete`）。
3. 调用删除器对象的 `operator()`，并将裸指针作为参数传入：`p2.get_deleter()(p2.get())`。

## 二、引入 Lambda 表达式作为删除器

### 代码：

```cpp
#include <iostream>
#include <memory>

struct XPacket {
    unsigned char* data = nullptr;
    int size = 0;
};

int main() {
		auto lambda_deleter = [](XPacket* p) {
        std::cout << "call Lambda Deleter()" << std::endl;
        std::free(p->data);
        delete p;
    };

    // 使用decltype推断lambda的类型，作为模板参数
    std::unique_ptr<XPacket, decltype(lambda_deleter)> p2 (new XPacket, lambda_deleter);
    return 0;
}
```

### 解析

#### 一、与类类型删除器 (`PacketDelete`)的区别

类类型删除器 (`PacketDelete`)

```cpp
std::unique_ptr<XPacket, PacketDelete> p1 (new XPacket); // 只需要传入指针
```