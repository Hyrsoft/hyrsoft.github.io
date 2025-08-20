---
title: Rust中的Enum
date: 2025-08-20 23:08:51
tags: Rust
categories: Programing
---
# Rust 中的枚举 (Enum)

枚举（Enumeration）是一种允许我们列举所有可能的值来定义一个类型的方式。

## 定义枚举

我们可以使用 `enum` 关键字来定义一个枚举类型，它的每个成员（变体）都可以代表一种可能的值。

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

### 将数据附加到枚举变体

Rust 的枚举一个强大的功能是，每个变体都可以关联不同类型和数量的数据。

```rust
enum IpAddrKind {
    V4(String),
    V6(String),
}

let home = IpAddrKind::V4(String::from("127.0.0.1"));
let loopback = IpAddrKind::V6(String::from("::1"));
```

我们还可以为每个变体定义不同类型和结构的关联数据。

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 }, // 关联一个匿名结构体
    Write(String),           // 关联一个 String
    ChangeColor(u8, u8, u8), // 关联一个三元组
}
```

## 为枚举实现方法

与结构体类似，我们也可以使用 `impl` 关键字为枚举定义方法。

```rust
impl Message {
    fn process(&self) {
        match self {
            Message::Quit => println!("Quit message received"),
            Message::Move { x, y } => println!("Move message received: x={}, y={}", x, y),
            Message::Write(text) => println!("Write message received: {}", text),
            Message::ChangeColor(r, g, b) => println!("ChangeColor message received: r={}, g={}, b={}", r, g, b),
        }
    }
}
```

## `Option` 枚举

`Option` 是一个由标准库定义的非常重要的枚举。它用于处理一个值可能存在或不存在的情况，有效地替代了其他语言中普遍存在的 `null`。

`Option<T>` 的定义如下：

```rust
enum Option<T> {
    Some(T), // 表示存在一个 T 类型的值
    None,    // 表示不存在值
}
```

使用 `Option` 可以让编译器强制我们处理值不存在的情况，从而提高代码的健壮性。

```rust
let x: i8 = 4;
let y: Option<i8> = Some(6);

// 为了处理 y 可能为 None 的情况，不能直接将 x 和 y 相加
// 必须使用 match 表达式来处理
match y {
    Some(val) => println!("x + y = {}", x + val),
    None => println!("y is None"),
}
```

## `match` 控制流运算符

`match` 是 Rust 中一个强大的控制流运算符，它允许我们将一个值与一系列模式进行比较，并根据匹配的模式执行相应的代码。`match` 常常与枚举一起使用，以确保我们处理了所有可能的变体。

`match` 的匹配是穷尽的，这意味着我们必须为所有可能的值提供一个分支。这在处理枚举时特别有用，因为编译器会检查是否遗漏了任何变体。

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```