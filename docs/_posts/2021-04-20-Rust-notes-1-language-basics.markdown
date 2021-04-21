---
layout: post
title:  "Rust 学习笔记（3）常见数据集合与错误处理"
date:   2021-04-21 10:24:54 +0800
categories: Rust Notes
---
# Rust 学习笔记三 -- 常用集合类型/错误处理

## 常用集合类型

集合：与内置的array和tuple不同，数据存储在堆上。常用的集合有：

- vector 线性存储结构，会把数据值按照内存临近的方式进行存储
- string 字符的集合
- hash map 存储关联的k-v对


Vector

- vec! 宏创建vector

String

- 一般指标准库中的String和string slice，但也有其他的String实现
- 不可以用下标读取，取slice的时候要注意一个char的长度

HashMap

- 可以使用 HashMap::new() 生成，也可以使用 vector 的 zip 命令生成
- 多次 insert 同一个key可以覆盖值
- for (k, v) in &map 可以对hashmap进行迭代
- map.entry(key).or\_insert(value) 可以实现当key不存在的时候进行插入，并且返回一个可变的引用
- 如果要依赖原有的值对value进行更新，可以使用 or\_insert 返回可变引用。之后使用 \*ref 进行变化计算


## 常见错误处理

错误处理

- 可恢复错误：使用 Result<T, E> 机制进行处理
- 不可恢复错误：使用panic! 宏进行处理

panic!

- --release 默认操作是打印日志，并且unwinding 栈数据，继续进行下去
- 在 Cargo.toml [profile.release] 加入 panic = 'abort' 可以设置panic的时候退出
- RUST\_BACKTRACE=1 环境变量可以设置打印出panic时候的调用栈信息

{% highlight rust %}
enum Result<T, E> \{
Ok(T),
Err(E),
\}
{% endhighlight %}


简化错误处理

- 使用unwrap\_or\_else 来简化大量的错误处理代码，即当为ok的时候可以执行unwrap，如果为err的时候执行 else 中的闭包处理错误
- 使用unwrap 和 expect 简化处理
    - unwrap 等价于 匹配Ok的时候返回包装值，匹配Err的时候，调用panic! 宏
        - let f = File::open("hello.txt").unwrap();
    - expect 与 unwrap 类似，不过可以指定更合适的报错信息
        - let f = File::open("hello.txt").expect("Failed to open hello.txt");

错误传播

- 场景：当我们的代码不知道上层业务该如何处理这种类型错误的时候，应当交给上层代码进行错误处理。这个叫做错误传播
    - 方法一：返回Result，当出现错误的时候返回 Err()
    - 方法二：使用 ? 操作符。如下代码，如果返回结果是Ok(f)，则会将f 赋值到变量上。如果返回结果是Err，则会直接将错误结果作为函数结果返回。
        - ? 操作符只能用在返回 Result 的函数内部，如果函数不返回Result，那只有两种方法：
            - 把接口格式改为返回Result
            - 使用 match 去处理错误

```
let mut f = File::open("hello.txt")?;

// 可以链式调用
File::open("hello.txt")?.read_to_string(&mut s)?;
```


main 函数特殊之处，除了返回空 () 之外，也可以返回Result

```
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box <dyn Error>> {
    let f = File::open("hello.txt")?;
    Ok(())
}
```



什么时机panic!

- Examples, Prototype Code, and Tests 适合 panic! 使得样例代码更加简洁易懂
- 当你比编译器有更多的信息可以判断不会出错的时候，比如 let home: IpAddr = "127.0.0.1".parse().unwrap();
- 当程序因错误参数会进入错误状态（非预期的，无法很好处理的状态）的时候，使用 panic! 去提醒外部调用者这种错误状态，进而避免这样调用

错误处理的一般指导意见：可预期的错误使用Result 返回，让调用者可以根据返回信息进行错误处理。

自定义类型参数校验

- 定义 new  方法，如果参数不可预期，则panic!，另外参数为私有，只提供公开的get方法。这样就不会通过其他途径创建类型实例使得参数不可预期



Reference:

1. [Common Collections](https://doc.rust-lang.org/book/ch08-00-common-collections.html)
1. [Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html)