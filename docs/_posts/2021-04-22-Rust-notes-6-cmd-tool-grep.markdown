---
layout: post
title:  "Rust 学习笔记（6）命令行小工具实践"
date:   2021-04-22 13:22:54 +0800
categories: Rust Notes
---


* 正确使用生命周期，比如在search 命令中，返回引用contents 中的内容，所以返回Vec 值中包含的 &str 引用的生命周期应该和 contents 保持一致。而不用和 query 保持一致。

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut result: Vec<&str> = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            result.push(line);
        }
    }
    result
}
```

* 使用 std::env 来处理环境变量和输入参数相关的内容

```rust
// 使用 std::env::args() 来获取命令行输入迭代器，使用collect 转成 Vec<String>
let args: Vec<String> = env::args().collect();

// 使用 std::env::var 来获取环境变量值
// 使用 is_err() 来判断环境变量没有设置
let case_sensitive = env::var("CASE_INSENSITIVE").is_err();
```

后续优化

通过把new 函数中的参数从&Vec 换成 env::Args 迭代器，原因在于

1. 传入 &[String] 的话，方法并没有args的所有权，所以想要将args[1] move 给query变量，必须使用clone 方法
2. 使用 env::Args 迭代器的话，在迭代过程中产生的参数，实际上所有权属于当前作用域，可以move到Config里面

```rust
// 之前，传入数组引用，当前作用域不具有所有权，无法move到Config里面，所以需要clone
pub fn new(args: &[String]) -> Result<Config, &str> {
    if args.len() < 3 {
        return Err("not enough arguments");
    }

    let query = args[1].clone();
    let filename = args[2].clone();

    let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

    Ok(Config {
        query,
        filename,
        case_sensitive,
    })
}


// 之后，传入迭代器，当调用next的时候才会生成对应的值，当前作用域完全具有所有权
// 所以可以把String 的所有权完全交给Config struct
pub fn new(mut args: env::Args) -> Result<Config, &'static str> {
    args.next();
    let query = match args.next() {
        Some(arg) => arg,
        None => return Err("Didn't get a query string")
    };
    let filename = match args.next() {
        Some(arg) => arg,
        None => return Err("Didn't get a file name")
    };
    let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

    let config = Config {
        query,
        filename,
        case_sensitive,
    };
    Ok(config)
}
```

参考链接

1. [Rust cmd tools practice](https://doc.rust-lang.org/book/ch12-00-an-io-project.html)