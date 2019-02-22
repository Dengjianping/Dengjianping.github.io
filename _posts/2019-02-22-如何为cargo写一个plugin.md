---
layout: post
title: "如何为cargo写一个插件(plugin)"
author: Jamie
---

## 如何为cargo写一个插件(plugin)

如果大家用过rust里的cargo工具，应该会觉得这大概是世界上最好的工程管理工具了吧。

我没有歧视python的pip/easy_install/pipenv, 或者c/c++的makefile/cmake的意思。

具体cargo的使用，可以参见官网 [**cargo**](https://doc.rust-lang.org/cargo/index.html)

目前，cargo不仅支持内建(built-in)的子命令，比如最常用的：
- **cargo new**
- **cargo build**
- **cargo run**

cargo具备良好的可可扩展性，也支持第三方的插件工具，下面罗列了些比较常用的cargo plugin。
- **cargo-expand**。在编译时生成标准macro和继承宏(derive，过程宏的一种)展开信息，这可以便于调试你所编写的宏。
- **cargo-update**。更新你所安装的cargo插件。
- **cargo-web**。这个可以帮助编译webassembly，比如配合[**yew**](https://github.com/DenisKolodin/yew)来使用，非常方面，还能即时编译当代码发生变动时。
- **cargo-graph**。生成工程的依赖关系图。
- **cargo-deb**。把当前工程编译成deb包，Debian/Ubuntu。
- **clippy**。这个应该是最常用的了。帮助大家写出更好更快的rust代码了。
- ...

这些插件都可以通过如下命令来安装到本地，比如：
```
$ cargo install cargo-web
```
官方也用心地罗列了[**crates.io**](https://crates.io/)上可用的第三方plugin。

[**cargo-plugin**](https://github.com/rust-lang/cargo/wiki/Third-party-cargo-subcommands)

这篇文章，就来简单介绍下如何为cargo写一个plugin。
这方面的知识，官方讲的不多，但并不意味着这有什么难度。具体文档看如下的链接。

[**custom-subcommands**](https://doc.rust-lang.org/cargo/reference/external-tools.html#custom-subcommands)


接下来将做一个最简单的例子，这个plugin能够显示当前的时间。比如
```
$ cargo date // 2019-02-21
```
1. 首先创建一个project。工程名字无所谓，主要是要注意cargo.toml的bin配置。
```
$ cargo new --bin cargo-date
```
2. 配置cargo.toml。把下面的这段加入到cargo.toml文件里。注意名字，格式一定要是**cargo-${command}**
```
[[bin]]
name = "cargo-date"
path = "src/main.rs"

[dependencies]
chrono = "0.4" # 3rd-party time module
```
3. 功能部分。这个很简单无需多言。为什么不用标准库的time模块，因为那个太简陋了。
```
use chrono::Local;

fn main() {
    let date = Local::now();
    println!("{}", date.format("[%Y-%m-%d][%H:%M:%S]"));
}
```
4. 编译并安装。这是最重要的一步。
```
// go to directory including cargo.toml
$ cargo install --path .
```
这条命令会编译并安装，安装目录分OS不同而不同。
- Windows: ```C:\Users\User_name\.cargo\bin\cargo-date```
- Linux: ```/home/user_name/.cargo/bin/cargo-date```

现在我们可以执行plugin了。
```
$ cargo date
```
这会打印如下信息
```
[2019-02-21][13:19:54]
```

所以大家看到了，编写cargo插件如此简单。当然了，如果要编写复杂的plugin，可能得用到这个crate。

[**clap**](https://clap.rs/)

Enjoy it!