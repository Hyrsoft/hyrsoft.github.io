---
title: Rust闭包
date: 2025-11-01 23:24:05
tags: Rust
categories: Programming
---
# Rust 闭包（Closure）

闭包是一种可以捕获其环境中变量的匿名函数。它在 Rust 中被广泛用于函数式编程、迭代器处理和并发编程。

下面的示例代码将围绕一个衬衫库存`Inventory`结构展开，演示闭包在实际场景中的应用。

```rust
use std::time::Duration;
use std::thread;

#[derive(Debug, PartialEq, Copy, Clone)]
enum ShirtColor {
    Red,
    Blue,
}

// 库存，表示有多少件不同颜色的衬衫
struct Inventory {
    shirts: Vec<ShirtColor>,
}

impl Inventory {
    // 根据用户偏好选择衬衫颜色，如果没有偏好则选择库存最多的颜色
    fn giveaway(&self, user_preference: Option<ShirtColor>) -> ShirtColor {
        // unwrap_or_else 在 user_preference 为 None 时调用闭包
        // 闭包返回值为T，即 most_stocked() 的返回值
        user_preference.unwrap_or_else(|| self.most_stocked())
    }

    fn most_stocked(&self) -> ShirtColor {
        let red_count = self
            .shirts
            .iter()
            .filter(|&&c| c == ShirtColor::Red)
            .count();
        let blue_count = self
            .shirts
            .iter()
            .filter(|&&c| c == ShirtColor::Blue)
            .count();
        if red_count >= blue_count {
            ShirtColor::Red
        } else {
            ShirtColor::Blue
        }
    }
}

fn main() {
    let store = Inventory {
        shirts: vec![ShirtColor::Blue, ShirtColor::Red, ShirtColor::Blue],
    };

    let user_pref1 = Some(ShirtColor::Red);
    let giveaway1 = store.giveaway(user_pref1);
    println!(
        "User preference: {:?}, Given away: {:?}",
        user_pref1, giveaway1
    );

    let user_pref2 = None;
    let giveaway2 = store.giveaway(user_pref2);
    println!(
        "User preference: {:?}, Given away: {:?}",
        user_pref2, giveaway2
    );
    
    // ... 后续代码 ...
}
```

在 `giveaway` 方法中，`unwrap_or_else` 接受一个闭包 `|| self.most_stocked()`。这个闭包没有参数（由 `||` 表示），并在被调用时执行 `self.most_stocked()`。

-----

### 1\. 闭包的类型推断与标注

Rust 编译器通常可以为闭包推断参数和返回值的类型。

```rust
// ... main 函数中 ...

// 1. 为闭包的参数和返回值显式标注类型
let expensive_closure = |num: u32| -> u32 {
    println!("This is an expensive closure!");
    thread::sleep(Duration::from_secs(2));
    num
};
println!("打印闭包 {:?}", expensive_closure(32));

// 2. 编译器自动推断类型
// 必须要求闭包的参数类型在上下文中可以被推断出来
let example_closure = |x| x;
let s = example_closure(String::from("hello"));

// 类型一旦被推断为 String，就不能再传入其他类型
// let n = example_closure(5); // 这将导致编译错误
```

### 2\. 捕获环境（借用与所有权）

闭包的核心特性是它可以捕获其定义时所在作用域中的变量。捕获方式有三种：不可变借用、可变借用和取得所有权。

#### 2.1. 不可变借用 (Fn)

闭包默认以最轻量的方式（不可变借用）捕获变量。

```rust
// ... main 函数中 ...
let list = vec![1, 2, 3];
println!("Before defining closure: {:?}", list);

// 闭包不可变地借用了 list
let only_borrows = || println!("From closure: {:?}", list); 

// 调用闭包
only_borrows();

// 闭包调用后，list 仍然可用
println!("After calling closure: {:?}", list);
```

#### 2.2. 可变借用 (FnMut)

如果闭包需要修改捕获的变量，它会进行可变借用。

```rust
// ... main 函数中 ...
let mut list2 = vec![1, 2, 3];
println!("Before defining closure: {:?}", list2);

// 闭包可变地借用了 list2
let mut borrows_mutably = || list2.push(7);

// 可变借用在闭包调用（borrows_mutably()）之前都有效
// 在此期间（闭包定义后到调用前）无法使用被借用的值
// println!("Before calling closure: {:?}", list2); // 错误：不能在可变借用期间再次借用

borrows_mutably(); // 调用闭包，可变借用在此处使用

// 调用闭包之后，可变借用结束，后续可以正常打印
println!("After calling closure: {:?}", list2);
```

#### 2.3. 取得所有权 (FnOnce 与 `move`)

使用 `move` 关键字可以强制闭包获取其使用的值（捕获的值）的所有权。这在并发编程中尤其重要，例如将变量传递给新线程。

```rust
// ... main 函数中 ...
let list3 = vec![1, 2, 3];
println!("Before defining closure: {:?}", list3); 

// 在闭包定义前加 move，表示闭包取得 list3 的所有权
thread::spawn(move || println!("From thread: {:?}", list3))
    .join()
    .unwrap();
    
// list3 的所有权已被移动到线程中，主线程无法再访问
// println!("After thread: {:?}", list3); // 错误：value borrowed here after move
```

*注意：新线程捕获主线程的值时，通常必须获取所有权（使用 `move`），否则可能在主线程结束后，新线程持有的引用变成悬垂引用。*

-----

### 3\. 闭包的 Trait：`Fn`、`FnMut`、`FnOnce`

Rust 根据闭包如何捕获和使用环境变量，自动为其实现一个或多个 `Fn` Trait：

| Trait | 捕获方式 | 描述 |
| :--- | :--- | :--- |
| `FnOnce` | 取得所有权 | 闭包必须被调用（消费）一次。所有闭包都至少实现了 `FnOnce`。 |
| `FnMut` | 可变借用 | 闭包可以修改捕获的变量，可以被调用多次。 |
| `Fn` | 不可变借用 | 闭包不能修改捕获的变量，可以被调用多次。 |

*继承关系：`Fn` 继承自 `FnMut`，`FnMut` 继承自 `FnOnce`。*

`Option<T>` 的 `unwrap_or_else` 方法的签名展示了 `FnOnce` 的应用：

```rust
// impl<T> Option<T> {
//     pub fn unwrap_or_else<F>(self, f: F) -> T
//     where
//         F: FnOnce() -> T, // 要求F实现FnOnce，表示闭包可以被调用一次
//     {
//         match self {
//             Some(val) => val,
//             None => f(), // 只有在None的情况下才调用闭包f
//         }
//     }
// }
```

### 4\. `FnMut` 与 `sort_by_key` 示例

`sort_by_key` 方法需要一个实现了 `FnMut` 的闭包，因为它在排序过程中可能需要多次调用闭包来获取键值。

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

// ... main 函数中 ...
let mut list_rect = [
    Rectangle { width: 10, height: 5 },
    Rectangle { width: 20, height: 10 },
    Rectangle { width: 30, height: 15 },
];

// 闭包 |r| r.width 实现了 FnMut (实际上也实现了 Fn)
list_rect.sort_by_key(|r| r.width); 
println!("Sorted by width: {:?}", list_rect);

// --- 演示 FnMut ---
let mut sort_operations = vec![];
let value = String::from("closure called"); // 稍后将被克隆
let mut num_sort_operations = 0; // 将被可变借用

list_rect.sort_by_key(|r| {
    // 闭包内部修改了捕获的变量 num_sort_operations，因此它必须是 FnMut
    sort_operations.push(value.clone()); // value 被克隆，所有权未移动
    num_sort_operations += 1; 
    r.width
});

println!("Sort operations: {}, Count: {}", sort_operations.len(), num_sort_operations);
```

