---
title: Rust智能指针
date: 2025-11-01 23:24:39
tags: Rust
categories: Programming
---
# 智能指针

智能指针（Smart Pointers）是类似指针的数据结构，但它们具有额外的元数据和功能。与引用（`&`）不同，引用仅借用数据，而智能指针通常拥有它们指向的数据。

常见的智能指针包括：

  * `Box<T>`
  * `Rc<T>`
  * `RefCell<T>`
  * `Ref<T>`
  * `Mutex<T>`
  * `Arc<T>`

-----

### 1\. `Box<T>`：在堆上分配数据

`Box<T>` 智能指针用于将数据存储在堆（Heap）上，而不是栈（Stack）上。它提供了数据的所有权，并在 `Box<T>` 离开作用域时自动释放堆上的数据。留在栈中的只是指向堆数据的指针。

`Box<T>` 的一个常见用途是构建递归数据结构（如链表），因为它可以帮助 Rust 确定结构的大小。

```rust
// 示例：使用 Box 构建递归的 List
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main_box() {
    // b 在栈上，但值 5 在堆上
    let b = Box::new(5); 
    println!("Boxed value: {}", b);

    let x = 5;
    // 将 x 的值（5）复制到堆上
    let y = Box::new(x); 

    assert_eq!(5, x);
    // 使用 * 解引用操作符来访问堆上的数据
    assert_eq!(5, *y); 
}
```

-----

### 2\. `Deref` Trait：自定义解引用行为

`Deref` Trait 允许我们自定义解引用操作符（`*`）的行为，使其能够像常规指针一样被解引用。

实现 `Deref` Trait 只需要实现 `deref` 方法：

```rust
use std::ops::Deref;

// 自定义一个类似 Box 的结构
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

// 为 MyBox 实现 Deref
impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        // 返回元组结构中的第一个元素 (T) 的引用
        &self.0
    }
}
```

#### Deref 强制转换 (Deref Coercion)

Deref Coercion（解引用强制转换）是 Rust 的一项便利特性。如果一个类型 `A` 实现了 `Deref<Target = B>`，那么 Rust 会在需要时自动将 `&A` 转换为 `&B`。

  * 例如：`String` 实现了 `Deref<Target = str>`，所以 `&String` 可以自动转换（强制转换）为 `&str`。

这同样适用于我们自定义的 `MyBox<T>`：

```rust
// (接上文 MyBox 的定义)

fn hello(name: &str) {
    println!("Hello, {}!", name);
}

fn main_deref_coercion() {
    let m = MyBox::new(String::from("Rust"));

    // Rust 自动执行了 &MyBox<String> -> &String -> &str
    hello(&m); 
}
```

-----

### 3\. `Drop` Trait：自定义清理逻辑

`Drop` Trait 允许我们在值离开作用域时，自定义需要执行的清理逻辑（类似于其他语言的析构函数）。

实现 `Drop` Trait 只需要实现 `drop` 方法：

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        // 定义清理逻辑
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main_drop() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    let d = CustomSmartPointer {
        data: String::from("other stuff"),
    };
    println!("CustomSmartPointers created.");
    // 当 main 结束时，变量会按其入栈顺序的逆序被 Drop
    // d 先被 drop，然后 c 被 drop
}
```

#### 提前释放资源 `std::mem::drop`

我们不能手动调用 `drop` 方法（例如 `c.drop()`）。如果需要提前释放资源，必须使用标准库提供的 `std::mem::drop` 函数。

`std::mem::drop` 会接管值的所有权，并在该函数结束时自动调用其 `Drop::drop` 方法，这可以防止双重释放（Double Free）错误。

```rust
// (接上文 CustomSmartPointer 的定义)

fn main_mem_drop() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    println!("CustomSmartPointer c created.");
    
    // 提前释放 c
    std::mem::drop(c); 
    
    println!("CustomSmartPointer c dropped before end of main.");
}
```

-----

### 4\. `Rc<T>`：引用计数

`Rc<T>` (Reference Counting) 是一种引用计数智能指针，它允许多个所有者（Multiple Ownership）共同拥有同一份堆上的数据。

  * 它会跟踪指向数据的引用数量。
  * 只有当引用计数归零时，数据才会被清理。
  * `Rc<T>` **只可用于单线程场景**。

-----

### 5\. `RefCell<T>` 与内部可变性

`RefCell<T>` (Reference Cell) 提供了一种实现**内部可变性**（Interior Mutability）的模式。它允许我们在持有不可变引用（`&T`）的情况下，仍然能够修改内部的数据。

  * `RefCell<T>` **只适用于单线程场景**。
  * 它并不会绕过 Rust 的借用规则，而是将借用规则的检查从**编译时**推迟到**运行时**。
  * 通过 `borrow()` (获取不可变引用) 和 `borrow_mut()` (获取可变引用) 方法在运行时检查借用规则。
  * 如果违反了借用规则（例如，同时存在多个可变借用），程序将在运行时 `panic`。

#### 实践示例：`RefCell<T>` 用于模拟（Mocking）

在测试中，我们经常需要一个“模拟对象”（Mock Object）来记录其上发生了哪些调用。

在下面的例子中，`MockMessenger` 需要在 `send` 方法（该方法只接收 `&self`，即不可变引用）内部修改 `sent_messages` 向量。`RefCell` 使得这种内部可变性成为可能。

```rust
pub trait Messenger {
    // send 方法接收的是不可变引用
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
where
    T: Messenger,
{
    pub fn new(messenger: &'a T, max: usize) -> LimitTracker<'a, T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger
                .send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger
                .send("Warning: You've used up over 75% of your quota!");
        }
    }
}

// --- 测试模块 ---
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        // 使用 RefCell 包装需要“内部可变”的数据
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        // 尽管 send 接收的是 &self，我们依然可以修改内部数据
        fn send(&self, msg: &str) {
            // 1. borrow_mut() 在运行时获取可变引用
            // 2. 如果违反借用规则（比如已存在其他借用），这里会 panic
            self.sent_messages.borrow_mut().push(String::from(msg));
        }
    }

    #[test]
    fn it_sends_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        // 3. borrow() 在运行时获取不可变引用
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

