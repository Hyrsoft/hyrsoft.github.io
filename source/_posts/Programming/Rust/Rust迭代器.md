---
title: Rust迭代器
date: 2025-11-01 23:24:21
tags: Rust
categories: Programming
---
# Rust 迭代器 (Iterators)

Rust 的迭代器（Iterator）允许我们对一个序列（如 `Vec`）中的每个项依次执行某些操作。

迭代器一个核心特性是**惰性 (Lazy)**：在调用“消耗性”方法（consuming methods）来驱动迭代器之前，迭代器本身不会执行任何操作。

### 1\. 创建和使用迭代器

可以通过在集合（如 `Vec`）上调用 `iter()` 方法来创建一个迭代器。`for` 循环是使用迭代器最常见的方式之一。

```rust
fn main() {
    let v1 = vec![1, 2, 3, 4, 5];

    // v1.iter() 创建了一个迭代器
    let v1_iter = v1.iter();

    // for 循环会获取迭代器的所有权并自动调用 next()
    for val in v1_iter {
         // 因为 .iter() 产生的是不可变引用，所以 val 的类型是 &i32
        println!("Got: {}", val);
    }
}
```

### 2\. Iterator Trait 与 next 方法

所有的迭代器都实现了标准库中的 `Iterator` Trait。这个 Trait 的核心是 `next` 方法。

  * `next()` 方法：
      * 每次调用 `next` 时，它会返回迭代器中的下一项，并将其包装在 `Some(T)` 中。
      * 当迭代器耗尽时，它会返回 `None`。
  * `next()` 方法会消耗（推进）迭代器的状态，因此如果手动调用 `next`，迭代器变量必须是可变的（`mut`）。
  * `for` 循环会自动处理迭代器的可变性。

<!-- end list -->

```rust
#[test]
fn iterator_demonstration() {
    let v1 = vec![1, 2, 3, 4, 5];

    // 当手动调用 next() 时，必须将迭代器声明为 mut
    let mut v1_iter = v1.iter();

    // next() 返回的是 Option<&T>
    assert_eq!(v1_iter.next(), Some(&1));
    assert_eq!(v1_iter.next(), Some(&2));
    assert_eq!(v1_iter.next(), Some(&3));
    assert_eq!(v1_iter.next(), Some(&4));
    assert_eq!(v1_iter.next(), Some(&5));
    assert_eq!(v1_iter.next(), None);
}
```

### 3\. 迭代器种类 (iter, iter\_mut, into\_iter)

根据创建方式的不同，迭代器可以产生不同类型的项：

| 方法 | 返回的项 | 描述 |
| :--- | :--- | :--- |
| `iter()` | `&T` | 创建一个返回不可变引用的迭代器。 |
| `iter_mut()` | `&mut T` | 创建一个返回可变引用的迭代器。 |
| `into_iter()`| `T` | 创建一个获取集合所有权的迭代器，返回项本身（`T`）。 |

### 4\. 消耗性适配器 (Consuming Adapters)

消耗性适配器（Consuming Adapters）是迭代器上的方法，它们会调用 `next` 来驱动迭代器，并最终消耗（使用掉）它。

`next` 方法本身就是一个消耗性适配器。

**示例：`sum`**
`sum` 方法会获取迭代器的所有权，遍历所有元素并计算总和。

```rust
#[test]
fn iterator_sum() {
    let v1 = vec![1, 2, 3, 4, 5];

    let v1_iter = v1.iter();

    // sum() 会获取 v1_iter 的所有权并消耗它
    let total: i32 = v1_iter.sum(); 
    // 调用 sum 之后，v1_iter 不能再被使用

    assert_eq!(total, 15);
}
```

**示例：`collect`**
`collect` 方法会消耗迭代器，并将结果收集到一个新的集合中（如下文 `map` 示例所示）。

### 5\. 迭代器适配器 (Iterator Adapters)

迭代器适配器（Iterator Adapters）是 `Iterator` Trait 上的方法，它们**返回一个新的迭代器**。它们是惰性的，在调用消耗性适配器之前不会执行任何操作。

**示例：`map`**
`map` 方法接收一个闭包，对迭代器中的每个元素调用该闭包，并返回一个包含新结果的新迭代器。

```rust
fn main_map() {
    let v2 = vec![1, 2, 3, 4, 5];
    
    // .iter() 是迭代器
    // .map() 是迭代器适配器 (惰性的)
    // .collect() 是消耗性适配器 (执行操作)
    let v3: Vec<_> = v2.iter().map(|x| x + 1).collect();
    
    assert_eq!(v3, vec![2, 3, 4, 5, 6]);
}
```

### 6\. 使用闭包捕获环境

许多迭代器适配器（如 `map`、`filter`）都接收闭包作为参数。闭包可以捕获其定义时所在的环境变量。

**示例：`filter`**
`filter` 方法接收一个闭包（返回 `true` 或 `false`），并返回一个只包含原迭代器中使闭包返回 `true` 的元素的新迭代器。

在下面的示例中，`filter` 闭包 `|s| s.size == shoe_size` 捕获了 `shoes_in_size` 函数参数中的 `shoe_size` 变量。

```rust
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

// 找出符合条件的鞋子（size 等于 shoe_size）
fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter() // 获取所有权，产生 T (Shoe)
        .filter(|s| s.size == shoe_size) // 闭包捕获了 shoe_size
        .collect() // 消耗迭代器，收集结果
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn filters_by_size() {
        let shoes = vec![
            Shoe { size: 10, style: String::from("sneaker") },
            Shoe { size: 13, style: String::from("sandal") },
            Shoe { size: 10, style: String::from("boot") },
        ];

        let in_my_size = shoes_in_size(shoes, 10);

        assert_eq!(
            in_my_size,
            vec![
                Shoe { size: 10, style: String::from("sneaker") },
                Shoe { size: 10, style: String::from("boot") },
            ]
        );
    }
}
```