---
title: Rust中的Slice
date: 2025-08-20 23:07:41
tags: Rust
categories: Programing
---
# Rust 中的字符串切片 (String Slice)

字符串切片（String Slice）是对 `String` 中一部分的引用。它允许你在不获取整个 `String` 所有权的情况下，引用其中的一部分连续数据。

## 创建切片

你可以使用方括号和一个范围 `[start..end]` 来创建一个字符串切片。

- `start`: 范围的起始索引，包含在切片中。
- `end`: 范围的结束索引，**不**包含在切片中。

```rust
let s = String::from("hello world");

// "hello" 的切片，索引从 0 到 4
let hello = &s[0..5]; 

// "world" 的切片，索引从 6 到 10
let world = &s[6..11]; 
```

范围语法有一些方便的简写：

| 语法          | 描述                      | 示例 (基于 `let s = String::from("hello");`) |
| ------------- | ------------------------- | -------------------------------------------- |
| `&s[..end]`   | 从开头到索引 `end` (不含) | `let slice = &s[..2];` // "he"               |
| `&s[start..]` | 从索引 `start` 到结尾     | `let slice = &s[3..];` // "lo"               |
| `&s[..]`      | 整个字符串的切片          | `let slice = &s[..];` // "hello"             |

## 切片的本质

切片本质上是一种特殊类型的引用，它是一个“胖指针”（fat pointer）或“宽指针”（wide pointer）。它内部存储了两部分信息：

1. 指向数据起始位置的指针。
2. 切片的长度。

## 将切片作为函数参数

将字符串切片 `&str` 作为函数参数，可以使函数更具通用性和灵活性。这样的函数既可以接收 `String` 的引用，也可以接收字符串字面量（它本身就是切片类型）。

```rust
// 这个函数可以接收 String 类型 (例如 &s) 和 &str 类型 (例如 "hello world")
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[..i]; // 返回直到第一个空格的切片
        }
    }
    
    &s[..] // 如果没有空格，返回整个字符串的切片
}
```

**调用示例**：

```rust
let my_string = String::from("hello world");

// first_word 对 &String 同样有效
let word = first_word(&my_string);
println!("第一个单词是: {}", word);

let my_string_literal = "hello world";

// first_word 对字符串字面量也有效
let word = first_word(my_string_literal);
println!("第一个单词是: {}", word);
```