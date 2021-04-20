---
layout: post
title:  "Rust 学习笔记（1）语言基础"
date:   2021-04-20 14:10:54 +0800
categories: Rust Notes
---

Ownership 的规则

* Rust中每个值都有一个变量是它的owenr -- Each value in Rust has a variable that’s called its owner.
* 每个时刻只能有一个owner -- There can only be one owner at a time.
* 如果owner已经离开了作用域，那么值会被释放掉 -- When the owner goes out of scope, the value will be dropped.

数据和变量的交互

* Move 当赋值发生的时候，值会move到另一个变量上。
* 如果是堆上的数据 如 let s1=String::from(“a”); let s2 = s1; 那么s1 值在move 到s2上之后，s1 就无法使用了。并且离开s2 的作用域，会调用值的 Drop 方法释放内存。
* Clone 如果实现了 clone 方法，则可以调用它实现类似其他语言深拷贝的语义
* Copy 栈上的值，发生赋值move后，原来的变量依然可用，是因为其有Copy 的annotation。
* Copy 的trait 中不包含任意部分的 含有 Drop 的trait 。如  (i32 , i32) 是 Copy trait，但是 (i32, String) 不是

Ownership 和 函数

* 栈上数据（Copy）类型作为参数传入函数，会用Copy方式传入，在离开函数调用后，在父级函数依然可用
* 堆上数据（Drop）类型作为参数传入函数，会转移ownership，move到函数里面，离开函数调用后，会触发Drop操作，所以在父级函数就无法再使用了
* 函数返回值会触发move 操作
* 如果把数据返回回去，则父函数可以获取并且是继续使用

Reference 和 Borrowing

* 在函数参数中使用引用，我们称之为Borrowing。参数分为 
    * 只读引用  
    * 可变引用
* 为了避免数据竞态（data race）
    * 同一时刻一个变量只能有一个mutable 引用
    * 同一时刻一个变量不能同时有mutable引用和immutable 引用

{% highlight rust %}
// Immutable Reference
fn calculate_length(s: &String) -> usize {
    s.len()
}

// Mutable Reference
fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
{% endhighlight %}

Slice
* string slice 是对于String 部分的引用 `let slice = &s[0..2];`
* 如果 slice 引用的原始 String，由于slice 不可变引用了 String，则String 在slice引用没有释放掉的时候，不能调用mutable方法

Struct
* Named Struct
    * 变量名字如果和字段名字相同，创建实例的时候可以直接去掉字段名
    * User { email : email } --> User { email }
    * 可以通过一个实例的值去创建另一个实例
* Tuple Struct
    * 字段只有类型，而没有名字。适合于为了取一个名字，区分不同的tuple
* Unit Struct
    * 没有任何字段，用来只是为了实现一些没有任何数据字段的 trait

{% highlight rust %}
// Instantiate an instance with another instance
let user2 = User {
    email : String::from(“abc@abc.com”),
    ..user1
}
{% endhighlight %}



Method syntax
* Method 类似function，唯一不同的是它是定义在struct/enum/trait 对象的上下文里面。类似其他语言的成员方法，实例用 . 调用
* associated functions 可以定义关联函数，但是他们不引用self，即不需要实例即可以引用。associated functions 类似其他语言的类型静态方法，类型用 :: 调用
* self 引用可以是只读引用，也可能是可变引用，用 &mut  表示

Enum
* 定义Enum 类似于为每一个枚举值定义一个Struct 类型。不同的是，Enum 可以把多个类似Struct类型（枚举值）封装在一个Enum类型中
* 可以为Enum 定义类似Struct 的方法

Pattern Matching
* Pattern matching, match 关键词接收一个Enum变量，分支是所有枚举值，如果枚举值有遗漏会产生编译错误。使用 _ 作为占位符表达不匹配其他分支时候的操作
* if let 命令，可以简化match操作，等价于match的第一个分支。另外也可以配合else使用，等价于 match 的 _ 统配匹配

{% highlight rust %}
if let Some(3) = some_u8_value {
    println!("three");
}
{% endhighlight %}


Reference:

1. [Rust-lang the book](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)