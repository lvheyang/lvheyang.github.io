---
layout: post
title:  "[Rust] 泛型和生命周期"
date:   2021-04-21 21:11:54 +0800
categories: Rust Notes
---

## 泛型

使用位置

* 类型定义
* 枚举类型定义
* 函数/方法定义

```rust
// 实现Point类型T泛型方法
impl<T> Point <T> {
  fn x(&self) -> &T {
    &self.x
  }
}

// 单一实现一种类型方法
impl Point<f32> {
  fn distance_from_origin(&self) -> f32 {
    (self.x.powi(2) + self.y.powi(2)).sqrt()
  }
}

// 实现Point 类型 <T, U> 和不同Point 类型 <V, W> 混合
// 需要不同类型签名区分
impl<T, U> Point<T, U> {
  fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
    Point {
      x: self.x,
      y: other.y,
    }
  }
}

fn main() {
  let p1 = Point { x: 5, y: 10.4 };      // Point<i32, f64>
  let p2 = Point { x: "Hello", y: 'c' }; // Point<String, char>

  let p3 = p1.mixup(p2);                 // Point<i32, char>

  println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

性能：

* Rust 使用 Monomorphization 单体化，在编译时会将代码填充每种具体类型后编译。所以运行时没有额外开销。

```
let integer = Some(5);
let float = Some(5.0);
```

如以上代码，编译器会检查，Option 初始化实例的时候，接受了 i32 和 f64 两种，则将这两种类型都填充到 Option 源码，编译成近似 Option_i32 和 Option_f64 两个具体类型。


生命周期检查

* 主要目的：避免野引用，即引用还在，原来的变量没了
* Rust 编译器提供Borrow Checker的功能，能够判断变量的生命周期，如果变量r 生命周期 ‘a ，变量x 生命周期 ‘b。其中 ‘a 长于  ‘b，而 r引用 x （let r = &x）。则会报编译错误。
* 即 引用变量 的生命周期 必须要短于（包含于）原变量的生命周期。

```rust
// 错误的示例，原变量x ownership borrow给 r，但是r 生命周期比x 长。则编译错误
{
  let r;                // ---------+-- 'a
                        //          |
  {                     //          |
      let x = 5;        // -+-- 'b  |
      r = &x;           //  |       |
  }                     // -+       |
                        //          |
  println!("r: {}", r); //          |
}                       // ---------+
```

显式生命周期声明

* 生命周期一般是 `'a`
* 单个生命周期一般没有什么意义，生命周期更多的是表达不同变量的生命周期相对关系
* 显式声明生命周期不会改变任何引用的生命周期。只是用来告诉编译器borrow checker 不能违反相关的生命周期约束

```
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```

例如：

```rust
// 函数并不知道 x/y 变量的生命周期，只是约束 x/y/return 引用都有相同的生命周期
// 在实际调用的时候'a会被真实的声明周期替换，才真正做borrow checker
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  if x.len() > y.len() {
    x
  } else {
    y
  }
}

//！错误例子
// result 虽然和 string1 生命周期一样，但是因为result/string1/string2 因为longest生命周期约束应该保持一致。
// 所以result 生命周期长于 string2 会导致编译错误
fn main() {
  let string1 = String::from("long string is long");
  let result;
  {
    let string2 = String::from("xyz");
    result = longest(string1.as_str(), string2.as_str());
  }
  println!("The longest string is {}", result);
}
```

* 不可能出现返回引用的生命周期和任何函数参数的生命周期都不相关
* 这种情况只可能是返回引用是从函数内部创建的，这样原值离开函数后会被销毁，所以Rust 编译器不允许这种情况发生。
* 建议：如果在函数内部创建值，则返回拥有所有权的变量值，不能返回引用


在声明Struct类型时候可以声明生命周期，这样可以让字段持有引用

```rust
struct ImportantExcerpt<'a> {
  part: &'a str,
}
```


Lifetime Elision / 生命周期省略原则

* 如果代码中生命周期存在歧义，则会直接报编译错误，提醒开发者显式生命生命周期
* 在函数/方法参数上的被称为输入生命周期，在返回值上的被称为输出生命周期，编译器会顺序执行以下三个规则，如果结束时仍然有引用无法推断生命周期，则失败
    * 规则一：假设每个引用输入参数都有自己独立的生命周期
    * 规则二：如果只有一个输入生命周期，则这个生命周期会赋予所有返回引用的输出生命周期
    * 规则三：如果有多个输入生命周期参数，不过其中有一个为 &self 或者 &mut self ，即它是一个方法，那么 self 的生命周期参数会赋予所有的输出生命周期参数

规则三非常有用，几乎让方法都不用显式声明生命周期参数了，一个方法生命周期的例子：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_return_part(&self, announcement: &str) -> &str {
        println!("Attention {}", announcement);
        self.part
    }
}

// 根据规则1，方法拥有两个输入参数，各自有独立的生命周期 'a , 'b
// 根据规则3，方法有一个参数为&self，那么返回引用的生命周期和 &self 相同为 'a 
```


静态生命周期  'static

* 会保证引用一直在程序中保留
* 所有的字面字符串都可以声明为静态引用
* 请使用之前仔细思考是否需要在程序运行期间一直生效

混合泛型、生命周期参数、trait绑定的例子

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
  x: &'a str,
  y: &'a str,
  ann: T,
) -> &'a str
  where
    T: Display,
{
  println!("Announcement! {}", ann);
  if x.len() > y.len() {
    x
  } else {
    y
  }
}
```


