---
title: Rust中的Struct
date: 2025-08-20 23:08:30
tags: Rust
categories: Programing
---
# Rust 中的结构体 (Struct)

结构体（Struct）是一种自定义数据类型，允许你将多个相关的值组合在一起，并为每个值命名。

## 定义与实例化

使用 `struct` 关键字来定义一个结构体，并在花括号中为所有字段指定名称和类型。

```rust
// 定义一个结构体
struct User {
    name: String,
    age: u8,
    email: String,
}

// 创建结构体实例
let user1 = User {
    name: String::from("Alice"),
    age: 30,
    email: String::from("alice@example.com"),
};
```

### 结构体更新语法

当你想要基于另一个结构体实例来创建一个新实例，并且只更改其中少量字段时，可以使用 `..` 语法。这会使用未显式设置的字段的旧实例中的值。

```rust
let user2 = User {
    email: String::from("hello@gmail.com"),
    ..user1 // user1 的 name 和 age 字段会被用于 user2
};
```

## 元组结构体 (Tuple Structs)

元组结构体是一种类似于元组的结构体，其字段没有名称，只有类型。

```rust
struct UserTuple(String, u8, String);
struct UserTuple2(String, u8, String);

let user3 = UserTuple(String::from("Bob"), 25, String::from("bob@example.com"));
```

**注意**：即使两个元组结构体具有完全相同的字段类型，它们也是不同的类型。例如，`UserTuple` 和 `UserTuple2` 是不可互换的。

## 使用 `derive` trait 增加功能

我们可以通过 `#[derive]` 注解为结构体自动实现某些 trait。例如，`Debug` trait 允许我们使用 `{:?}` 或 `{:#?}` 格式化字符串来打印结构体实例，这对于调试非常有用。

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

let rect1 = Rectangle {
    width: 30,
    height: 50,
};

// 使用 Debug trait 进行打印
println!("{:#?}", rect1);
```

## 结构体方法

方法与函数类似，但它们在结构体（或枚举、trait对象）的上下文中定义，并且它们的第一个参数总是 `self`，代表调用该方法的结构体实例。

### 定义方法

方法在 `impl` (implementation) 块中定义。

```rust
impl Rectangle {
    // 方法的第一个参数永远是 self

    // 计算面积，该方法借用了 Rectangle 实例的不可变引用
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // 判断一个矩形能否容纳另一个矩形
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width >= other.width && self.height >= other.height
    }
    
    // 可变方法，借用了实例的可变引用，可以修改实例的字段
    fn set_width(&mut self, width: u32) {
        self.width = width;
    }
}
```

| `self` 类型 | 描述                                   |
| ----------- | -------------------------------------- |
| `&self`     | 对实例的不可变借用，用于只读操作。     |
| `&mut self` | 对实例的可变借用，用于修改实例的状态。 |
| `self`      | 获取实例的所有权，将其移出调用者。     |

调用可变方法：

可变方法必须由一个可变的结构体实例来调用。

```rust
let mut rect3 = Rectangle { width: 10, height: 20 };
rect3.set_width(15);
println!("Rectangle 3 的宽度是 {}", rect3.width);
```

### 关联函数 (Associated Functions)

`impl` 块中也可以定义不以 `self` 作为第一个参数的函数，这被称为关联函数。它们通常用作构造函数。

```rust
impl Rectangle {
    // 这是一个关联函数，因为它没有 self 参数
    // Self 是当前 impl 块中类型的别名，即 Rectangle
    fn square(size: u32) -> Self { 
        Self {
            width: size,
            height: size,
        }
    }
}
```

关联函数使用 `::` 语法调用，而不是 `.` 语法。

```rust
// 调用关联函数
let rect2 = Rectangle::square(20);
```