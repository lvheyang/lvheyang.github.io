---
layout: post
title:  "[Rust] 自动化测试"
date:   2021-04-22 10:10:54 +0800
categories: Rust Notes
---

单元测试的示例。可以使用 #[test] 注解标识需要运行的测试。可以使用宏来检验结果符合预期

* assert!
* assert_eq!
* assert_ne!

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }
}

// 使用cargo test 运行
// 可以使用 assert!(<表达式>, <字符格式串>, <...需要渲染的变量>)  来自定义报错
```

* 可以使用 #[should_panic] 注解检验必须要panic

```rust
// #[should_panic] 来检验函数必须panic
// should_panic 可以配置 expected 参数来匹配对应panic信息

#[test]
#[should_panic(expected = “expected panic message”)]
fn exploration() {
    method_will_panic(...)
}
```

* 可以使用 Result<(), String> 来写test case，如果返回Ok(()) 则测试通过，如果 Err(String) 则会失败。注意 #[should_panic] 不能和Result 类型case 共用。

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}

```

* cargo test 和 cargo test --，前者是测试的命令，后者 -- 用来分割 cargo test 本身的参数，和生成的测试程序接收的参数，比如

```
cargo test --help // 输出 cargo test 的命令参数
cargo test test_name // 运行名字中包含 test_name 的测试


cargo test -- --help // 输出 cargo test -- 的命令参数
cargo test -- --nocapture // 不阻止测试向stdout/stderr 输出
cargo test -- --show-output // 输出成功的case 的标准输出
cargo test -- --test-threads=1 // 使用一个线程运行单元测试
```

* 标注#[ignore] 来忽略某些case，在 cargo test 运行的时候会忽略，可以使用 cargo test -- --ignored 专门运行被忽略的case

```rust
#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```

单元测试

* 单元测试以模块内部为主，可以测试私有的方法和逻辑。一般会将单元测试代码放置在src 代码目录下，并且创建一个module 名为 tests ，并且加上 #[cfg(test)] 注解
* 使用cfg 代表被注解的模块只有在某种配置情况下才会被包括进来，比如test 的时候才会编译、运行这些代码。不仅仅包含标注为 #[test] 的测试代码，也包含tests模块里面定义的帮助方法
* 因为tests 和属于lib的内部模块，所以可以测试内部函数

```rust
fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}

```

集成测试

* 在src 平级目录下创建 tests 目录，这个目录下可以放任意多的测试文件。每个测试文件会成为一个test section

```
cargo test --test integration_test // 可以用--test 参数选择集成测试模块
```

* 集成测试内部的通用方法
* 【不推荐】第一种方法，可以在 tests 目录下写一个 common.rs 文件，写通用逻辑，然后在不同的test case中引用。缺点是：会在cargo test的时候生成一个test section 不包含任何case
* 【推荐】第二种办法，通过在 tests/common/mod.rs 写入通用逻辑，然后通过模块进行引用

binary 类型代码不能直接使用 tests 目录下的集成测试进行测试，所以我们一般将代码大量写在lib.rs 中，少量的 main.rs 代码只是调用 lib.rs 中的库代码。所以我们主要用集成测试去测试 lib.rs 的库。
