---
layout: post
title:  "Rust 学习笔记（2）模块系统"
date:   2021-04-20 21:01:54 +0800
categories: Rust Notes
---

# Rust 学习笔记二 -- 模块系统

Module System

- Packages: 包管理，是一种Cargo的特性，用来编译、测试和分享crates
- Crates: 树状的Module组织结构，生成 库项目 library 或者 可执行项目 executable
- Modules and use: 用来组织代码，控制organization、scope（作用域）、私有的paths等
- Paths: 命名item的方法，如struct、function、module 等


Packages

- 一个Crate 是一个binary 或者 一个 library
    - crate root 是 Rust 编译器开始和组织Crate 的 root module
    - src/main.rs 和 src/lib.rs 是 crate root
- 一个Package 包含一个或者多个 Crate ，提供一系列相关的功能集合
    - 每个package 必须要包含Cargo.toml 描述如何编译这些crate
- Package 和 Crate 的目录关系
    - 至少要包含一个crate，包含任意多个binary crate，包含0或1个library crate
    - src/main.rs src/lib.rs 是默认的binary和library crate，如果包含着两个，相当于package 包含两个crate
    - 可以在 src/bin 下面创建多个文件，每个文件都会是一个独立的 binary crate

Module: 控制作用域和数据可见性

关键字

- mod 声明一个module
    - 第一：可以在当前文件中使用 mod xxx \{\} 直接定义module中的内容
    - 第二：可以在当前目录下生成 xxx.rs 文件，在这个文件中定义module中的内容
    - 第三：可以在当前目录下生成 xxx/mod.rs 文件，在这个文件中定义module中的内容
- use 引用 path 到当前作用域
- pub 将项目设置为公开
- as 将path 重命名

Crate 层级结构

- src/main.rs 和 src/lib.rs 是 crate root。所有的module 构成一个树状结构，继承自隐含的 crate 节点

Paths: 引用Module 树中的其他Item

- 绝对路径：使用crate 名从crate root 引用，或者使用 crate 直接引用
- 相对路径：使用self、super等关键词从当前module 使用标识ID引用

将 Struct 和 Enum 设置为公开

- Struct 可以设置为公开，也可以选择设置某些字段为公开的
    - 如果 Struct 有某些字段是私有的话，必须声明相关的公开构造方法，不然没有办法初始化Struct 实例
- Enum 不能设置私有，如果Enum 被设置为公开的，则所有枚举类型都是公开的


Paths use

- 可使用use 关键字，将路径引入当前作用域，类似于文件系统的软连接
    - 引用函数的时候最佳实践是引入父级 路径 ，使用 parent::function 调用
    - 引用Struct、Enum 等的时候，一般最佳实践是直接引用全路径。如果引入的时候出现重名，则会引入父级路径
- pub use 可以达到 re-exporting 把其他的path 重新公开给外部项目
    - 常见的场景是把内部的一些path 按照外部项目的上下文环境重新导出
- 嵌套引用  同时可以引用多个path
    - use std::\{cmp::Ordering, io\};
    - use std::io::\{self, Write\}; 用self 指代当前module
- 通配引用 \* 引用所有内容


将module 分布在不同的文件中

- lib.rs 中定义 mod xxx;
- 可以在xxx.rs 中
    - 直接定义模块 pub mod yyy \{...\}
    - 可以仅仅定义模块 pub mod yyy; 在 src/xxx/yyy.rs 中定义模块





Reference:

1. [Rust-lang the book](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)