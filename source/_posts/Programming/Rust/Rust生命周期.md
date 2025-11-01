---
title: Rust生命周期
date: 2025-11-01 23:24:54
tags: Rust
categories: Programming
---
# Rust 生命周期 (Lifetime)

Rust 中的每一个引用都有其生命周期（lifetime），也就是引用保持有效的作用域。大部分时候，生命周期是隐式且可以被推断的，类似于类型推断。然而，当引用的生命周期可能以多种方式互相关联时，Rust 会要求我们使用泛型生命周期参数来手动标注生命周期，以确保运行时不会出现悬垂引用。

------

## 借用检查器与生命周期

生命周期的主要目标是避免悬垂引用，它会导致程序引用了非预期引用的数据。

考虑一个返回两个字符串切片中较长者的函数。下面的代码无法通过编译，因为 Rust 编译器无法在编译期知道返回的引用指向 `x` 还是 `y`，因此无法确定其生命周期。

```rust
// 无法编译：返回的引用的生命周期不明确
// fn longest(x: &str, y: &str) -> &str {
//     if x.len() > y.len() {
//         x
//     } else {
//         y
//     }
// }
```

> 错误信息：missing lifetime specifier
>
> 这个错误表明，编译器需要我们明确告知返回的引用的生命周期与输入参数的生命周期之间的关系。

------

## 泛型生命周期注解

为了解决上述问题，我们需要为函数签名添加泛型生命周期参数。生命周期注解不会改变任何引用的生命周期，而是描述多个引用生命周期之间的关系。

- 生命周期参数以撇号（`'`）开头，名称通常是小写字母，如 `'a`。
- `&'a str` 读作 “一个至少活得和生命周期 `'a` 一样长的字符串切片引用”。

通过添加泛型生命周期 `'a`，我们告诉编译器：输入参数 `x`、`y` 和返回的引用必须拥有相同的生命周期（即它们中生命周期最短的那个）。

```rust
// 添加泛型生命周期参数'a，表示返回值的生命周期与输入参数中生命周期较短的一个相同
fn longest_with_lifetime<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

在下面的 `main` 函数示例中，`string1` 的生命周期比 `string2` 长。当它们被传入 `longest_with_lifetime` 时，`'a` 的具体生命周期将等于 `string2` 的生命周期。因此，返回的 `result` 的生命周期也与 `string2` 相同。

```rust
fn main() {
    let string1 = String::from("abcd");
    {
        let string2 = String::from("xyz");
        let result = longest_with_lifetime(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    } // string2 在这里失效，result 的生命周期也到此结束
}
```

------

## 结构体定义中的生命周期注解

当结构体持有引用时，我们也必须在其定义中添加生命周期注解。这保证了该结构体的实例存活时间不会超过其包含的引用的存活时间。

`ImportantExcerpt` 结构体持有一个字符串切片 `part`。其定义中的生命周期注解 `'a` 表明，`ImportantExcerpt` 的实例不能比它所引用的 `part` 活得更久。

```rust
// Struct 中使用生命周期注解
// 指明 ImportantExcerpt 实例的生命周期不能超过 part 引用的生命周期
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();

    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

------

## 生命周期省略规则（Lifetime Elision Rules）

在实践中，为了方便，编译器内置了一套生命周期省略规则。如果代码符合这些规则，我们就不需要显式地标注生命周期。这些规则不是让编译器猜测，而是遵循一套明确的算法。

首先，定义两个术语：

- **输入生命周期**：函数或方法参数上的生命周期。
- **输出生命周期**：函数或方法返回值上的生命周期。

编译器使用以下三条规则来推断生命周期：

| **规则**   | **描述**                                                     |
| ---------- | ------------------------------------------------------------ |
| **规则 1** | 编译器为每个输入引用（函数或方法的参数）分配一个不同的生命周期参数。 |
| **规则 2** | 如果只有一个输入生命周期参数，那么该生命周期会被赋给所有输出生命周期参数。 |
| **规则 3** | 如果有多个输入生命周期参数，但其中一个是 `&self` 或 `&mut self`，那么 `self` 的生命周期会被赋给所有输出生命周期参数。 |

如果编译器应用完这三条规则后，仍然无法确定所有输出引用的生命周期，就会报错并要求手动标注。

------

## 方法定义中的生命周期

在为带有生命周期的结构体实现方法时，其语法与泛型类型参数相似。

- 需要在 `impl` 关键字后声明生命周期参数 `'a`。
- 这些生命周期参数是 `ImportantExcerpt` 类型的一部分。

```rust
impl<'a> ImportantExcerpt<'a> {
    // 规则 3 适用：输入生命周期是 &'a self，所以输出生命周期也被推断为 'a
    // 因此 &self 和返回的 &str 拥有相同的生命周期
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

由于生命周期省略规则（特别是第三条），`announce_and_return_part` 方法不需要显式标注生命周期。

------

## 静态生命周期 (`'static`)

`'static` 是一个特殊的生命周期，它表示引用在整个程序的运行期间都有效。所有的字符串字面量都拥有 `'static` 生命周期，因为它们被直接存储在程序的二进制文件中。

```rust
let s: &'static str = "I have a static lifetime.";
```

## 泛型、Trait Bound 与生命周期的结合

我们可以在一个函数签名中同时使用泛型类型参数、Trait Bound 和生命周期，以创建更灵活和强大的函数。

下面的函数 `longest_with_an_announcement` 综合了这些概念：

- `'a`：泛型生命周期参数，确保引用的有效性。
- `T`：泛型类型参数。
- `where T: Display`：Trait Bound 约束，要求类型 `T` 必须实现 `Display` trait，以便能被打印。

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
where
    T: Display, // Trait bound约束
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
