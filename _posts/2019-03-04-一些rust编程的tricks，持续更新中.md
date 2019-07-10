---
layout: post
title: "一些rust编程的tricks，持续更新中"
author: Jamie
---

任何一门编程语言都有自己的一些tricks，这能帮助你代码优雅，节省时间，提高性能等等。所以知道一门语言的tricks是非常有必要的，当你学习这门语言一段时间后。

我用rust有一段时间了，之前用python比较多，现在已经几个月没写过了，所以自己也有一些心得，也从别人那里学习到很多，博客，stackoverflow，reddit，rust官方论坛等等。

所以这篇文章就列举一些我所知道的rust编程tricks。
如果是老鸟，就可以忽略不看了。

### 2019/03/04

#### 枚举转换成数字(enum to number)

```rust
// 不过有限制，里面的变量必须全是空的，不能带数据
// enum E { A, B(String), C}, E::A as u32 则会报错
enum E { A, B, C}

// ...
let a = E::A as u32; // 0
let b = E::B as u32; // 1
let c = E::C as u32; // 2
```

#### 解构元祖或者数组(slice pattern)

```rust
let a = [1,2,3,4];
let b = (1,2,3,4);

match &a {
    &[first, second, _, _] => println!("slice pattern: {}, {}", first, second),
    _ => unreachable!()
}

match &b {
    &(first, second, _, _) => println!("slice pattern: {}, {}", first, second),
    _ => unreachable!()
}

// 比如在未知长度的数组转换成元祖，是有点麻烦的，但有切片模式就比较方便来
let c = match &a {
    &[first, second] => (first, second),
    &[first, second, third] => (first, second, third),
    &[first, second, third, fourth] => (first, second, third, fourth),
    &[first, second, third, fourth, fifth] => (first, second, third, fourth, fifth),
    // 可能还有别的长度
    // ...
    _ => println!("unknow the length of array"),
}
```

不过目前还是只能做一些基本的切片匹配，比如这种省略模式 ```&[first, second, ..]```。

在将来应该会提供更丰富的匹配功能。

#### 嵌套枚举解构(destruct nested enum)

```rust
// lots of defined enums
enum A{ a(String)}
enum B{b(A)}
enum C{c(B)}
enum D{d(C)}
enum E {
    e(D),
}

// ...
let r = "jdent".to_string();
let a = A::a(r);
let b = B::b(a);
let c = C::c(b);
let d = D::d(c);
let e = E::e(d);

// ...
// 方法比较笨，但目前貌似也没有好的捷径
match e {
    E::e(D::d(C::c(B::b(A::a(ref s))))) => {
        println!("detructed value: {}", s);
    }
    _ => println!("nana"),
}
```

#### 嵌套的Option<T> or Result<T, E>

```rust
// 可能还会嵌套更多
let n = Ok(Some(3));
if let Ok(Some(v)) = n {
    // ...
} else {
    // ...
}
```

#### 更新静态变量
目前，如果在一个函数里面定义了一个静态变量，只能用unsafe方式来更新，尽管有时候这完全是安全的，比如:

```rust
fn update_static_val() {
    static mut calling_times: u32 = 0;
    calling_times += 1;
    calling_times
}
```

你会得到如下类似错误:

```
calling_times += 1;
^^^^^^^^^^^^^^^^^^ use of mutable static
= note: mutable statics can be mutated by multiple threads: aliasing violations or data races will cause undefined behavior

error[E0133]: use of mutable static is unsafe and requires unsafe function or block
```

使用unsafe就能通过。

```rust
fn update_static_val() -> u32 {
    static mut calling_times: u32 = 0;
    unsafe {
        calling_times += 1;
        calling_times
    }
}
```

但其实还有另外一种选择:

```rust
use std::sync::atomic::{ AtomicUsize, Ordering };

// ...
fn alter_way() -> usize {
    static atomic_num: AtomicUsize = AtomicUsize::new(0);
    atomic_num.fetch_add(1, Ordering::SeqCst);
    atomic_num.load(Ordering::Relaxed)
}
```
文档:
- [**AtomicUsize**](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html)

唯一担忧的是二者性能上有差别没有，有待测试。

#### &Option<T> to Option<&T> / &Result<T， E> to Result<&T， &E>
假如有这样的一个结构体:

```rust
struct A {
    a: Option<String>,
    b: Result<String, String>,
}

// ...
let ref m = A{a: Some("option".to_string()), b: Ok("result".to_string())};
// 这样借用是会直接报错的
let c = m.unwrap();
```

错误信息可能如此:

```rust
let c = m.a.unwrap();
        ^^^ cannot move out of borrowed content
```

其实这时候可以用AsRef trait，Option/Result都实现了该trait:

```rust
let c = m.a.as_ref().unwrap();
let d = m.b.as_ref().unwrap();
```

所以这个时候不需要去用Clone trait，避免了重新分配内存而影响性能。

#### @ 和 ...

这其实也不算tricks，只是rust的一种稍显奇怪的表达方式。

```rust
let a = 2u32;

match a {
    b @ 1 ... 10 => println!(" {} is in this range", e),
    _ => println!("not in this range")
}

```

#### 整型进制转换

```rust
let a = 78u32;

// ...
// 不足会补全8位
let a_bin = format!("{:08b}", a); // 01001110
let a_hex = format!("{:08x}", a); // 0000004e
```

详情可见官方文档。
- [fmt](https://doc.rust-lang.org/beta/std/fmt/index.html)


#### 链式写法来处理Result/Option

有时候错误处理的模式匹配写法，会有点让代码略显冗长，但是用链式写法就会简洁些，但代码可读性会有所降低，比如
```rust
// let a = Some(2u32);
let a = None;
// 使用match
let b = match a {
    Some(val) => Some(2 * val),
    _ => None
};

// 链式
let b = a.map_or(None, |val: u32| Some(2 * val));

let a: Result<u32, i32> = Err(-1i32);
let b = a.map_err(|e| format!("error code {} happened", e));
```

#### 一些好用的标准宏
- [dbg!](https://doc.rust-lang.org/beta/std/macro.dbg.html)(start from 1.32)。调试宏，很有用，变量，表达式都可以。
- [concat!](https://doc.rust-lang.org/beta/std/macro.concat.html)。快速连接字符串，比如：```let s = concat!("test", 10, 'b', true);```
- [env!](https://doc.rust-lang.org/beta/std/macro.env.html)/[option_env!](https://doc.rust-lang.org/beta/std/macro.option_env.html)。快速获取环境变量，比如：```let path: &'static str = env!("PATH");```
- [include!](https://doc.rust-lang.org/beta/std/macro.include.html)/[include_bytes!](https://doc.rust-lang.org/beta/std/macro.include_bytes.html)/[include_str!](https://doc.rust-lang.org/beta/std/macro.include_str.html)。快速打开utf-8文件，比如：```let bytes = include_bytes!("spanish.in");```
- [file!](https://doc.rust-lang.org/beta/std/macro.file.html)/[line!](https://doc.rust-lang.org/beta/std/macro.line.html)/[column!](https://doc.rust-lang.org/beta/std/macro.column.html)/，分别显示当前文件名，行号，列号。

#### 用Default trait来初始化数据

比如这样的情况
```rust
struct A {
    m: String,
    n: u32,
    k: i32,
}

// ...
// 前提是，这些成员都是实现了Default trait
let b = A {n: 10, ..Default::default()};
```

#### 函数式编程的另一种写法

通常情况是这样的:

```rust
let a = vec![1.2f32, 2.3, 4.6];
let b: Vec<_> = a.iter().map(|v| v.floor()).collect();
```

但实际上，也可以这样写

```rust
// let b: Vec<_> = a.iter().map(f32::floor).collect();
// 不能写如上的代码，因为floor的函数签名是这样的 pub fn floor(self) -> f32,
// 但iter带迭代 &T，而into_iter是迭代 T，注意参数类型。
let b: Vec<_> = a.into_iter().map(f32::floor).collect();
```

还比如这样获取数组里字符串的长度:

```rust
let v_str = vec!["hello", "world", "rust"];

// 通常是这样的
let str_len: Vec<_> = v_str.iter().map(|s| s.len()).collect();

// 但实际上还可以这么写
// 多说一句，len的函数签名是 pub fn len(&self), 所以用 iter
let str_len: Vec<_> = v_str.iter().map(str::len).collect();
```

### 2019/07/09 更新

#### 巧用match和元祖

比如你有一段如下代码，这样写就稍显冗长。

```rust
let a = Some(10i32);
let b = Some("match");

if a.is_some() && b.is_some() {
    let _a = a.unwrap();
    let _b = b.unwrap();
    // do something
} else {
    // do something else
}
```

利用元祖就可以这样写，简洁优雅

```rust
match (a, b) {
    (Some(_a), Some(_b)) => // do something
    _ => // do something else
}
```

或者

```rust
if let (Some(_a), Some(_b)) = (a, b) {
    // do something
} else {
    // do something else
}
```

#### 错误处理

有时候引入第三方crate，而其不同API返回的错误类型就不一样，有时候处理错误很麻烦，比如这样

```rust
fn handle_error() -> Result<i32, /**/> {
    let a: i32 = "32".parse()?;
    let b = std::fs::File::open("file_path")?;

    // do something else
    Ok(a)
}
```

但其实可以这样做，

```rust
type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync + 'static>>;
// 涉及到多线程的错误处理，需要加上 Send + Sync

fn handle_error() -> Result<i32> {
    let a: i32 = "32".parse()?;
    let b = std::fs::File::open("file_path")?;
    
    // do something else
    Ok(a)
}
```

#### 统一使用 ？

学习过rust的都知道，rust的错误处理有个语法糖，就是 ？，非常简洁易用。
但如果一个函数内既有 ```Result<T, E>``` 和 ```Option<T>```，这个就不能直接使用 ？ 了。幸亏Result 和Option都提供内置的相互转换的函数。

```rust
type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync + 'static>>;
// ...

fn handle() -> Result<i32> {
    let a: i32 = "32".parse()?;
    let b: i32 = Some(12).ok_or("error happened")?; // Option to Result
    Ok(a + b)
}
```

当然也可以Result to Option.

```rust
fn handle() -> Option<i32> {
    let a: i32 = "32".parse().ok()?;
    let b: i32 = Some(12)?; // Result to Option
    Some(a + b)
}
```

#### std::convert::identity

有时候你可能有一个这样的数组，想要提取出不是None的元素

```rust
let a = vec![Some(32i32), None, Some(45), Some(100)];
let b: Vec<_> = a.iter().filter(Option::is_some()).collect(); // 其实这样也够简洁了
```

但巧用 std::convert::identity，也可以达到此目的

```rust
use std::convert::identity;

let a = vec![Some(32i32), None, Some(45), Some(100)];
let b: Vec<_> = a.iter().filter_map(identity).collect();
```

#### 根据编译条件决定是否要执行某些语句

目前，stable版本的rust还不支持控制单条语句的执行，只能在nightly版本体验，具体看这个链接：

https://github.com/rust-lang/rust/issues/15701

```rust
let mut a: i32 = 32;

#[cfg(feature = "add_3")]
a += 3;

#[cfg(feature = "add_5")]
a += 5;
```

若要在stable版本中实现，则需要加上 ```{}```，比如根据工程的feature来决定是否要编译一条语句。

```rust
#[cfg(feature = "add_3")]
{
    a += 3;
}

#[cfg(feature = "add_5")]
{
    a += 5;
}
```

这个我觉得还是挺有用的，希望快点稳定下来。


### 结语
文中提到的，如果熟悉rust的人，可能这都不是什么tricks了。但这篇文章还是会持续更新中，因为rust每次更新都会带来一些新的feature。

持续更新中！