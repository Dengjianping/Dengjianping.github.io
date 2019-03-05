---
layout: post
title: "谈一谈Fn, FnMut, FnOnce的区别"
author: Jamie
---

在Rust里，闭包被分为了三种类型，列举如下
- [**Fn(&self)**](https://doc.rust-lang.org/std/ops/trait.Fn.html)
- [**FnMut(&mut self)**](https://doc.rust-lang.org/std/ops/trait.FnMut.html)
- [**FnOnce(self)**](https://doc.rust-lang.org/std/ops/trait.FnOnce.html)

#### 三种闭包的定义

在标准库里是怎么定义的。
 
- **FnOnce**

```rust
#[lang = "fn_once"]
pub trait FnOnce<Args> {
    type Output;
    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
```

参数类型是 ```self```，所以，这种类型的闭包会获取变量的所有权，生命周期只能是当前作用域，之后就会被释放了。

例子:

```rust
#[derive(Debug)]
struct E {
    a: String,
}

impl Drop for E {
    fn drop(&mut self) {
        println!("destroyed struct E");
    }
}

fn fn_once<F>(func: F) where F: FnOnce() {
    println!("fn_once begins");
    func();
    println!("fn_once ended");
}

fn main() {
    let e = E { a: "fn_once".to_string() };
    // 这样加个move，看看程序执行输出顺序有什么不同
    // let f = move || println!("fn once calls: {:?}", e);
    let f = || println!("fn once closure calls: {:?}", e);
    fn_once(f);
    println!("main ended");
}
```

打印的内容差不多是这样的:

```
fn_once begins
fn once calls: E { a: "fn_once" }
fn_once ended
main ended E { a: "fn_once" }
destroyed struct E
```

但是如果闭包运行两次，比如:

```rust
fn fn_once<F>(func: F) where F: FnOnce() {
    println!("fn_once begins");
    func();
    func();
    println!("fn_once ended");
}
```

则编译器就报错了，类似这样:

```
   |
14 |     func();
   |     ---- value moved here
15 |     func();
   |     ^^^^ value used here after move
   |
   = note: move occurs because `func` has type `F`, which does not implement the `Copy` trait
```

这是为什么呢？

还是回到 **FnOnce** 的定义，参数类型是 ```self```，所以在 ```func``` 第一次执行完之后，之前捕获的变量已经被释放了，所以已经无法在执行第二次了。如果想要运行多次，可以使用 **FnMut/FnOnce** 。

- **FnMut**

```rust
#[lang = "fn_mut"]
pub trait FnMut<Args>: FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}
```

参数类型是 ```&mut self```，所以，这种类型的闭包是可变借用，会改变变量，但不会释放该变量。所以可以运行多次。

和上面的例子差不多，有两个地方要改下:

```rust
fn fn_mut<F>(mut func: F) where F: FnMut() {
    func();
    func();
}

// ...
let mut e = E { a: "fn_once".to_string() };
let f = || { println!("FnMut closure calls: {:?}", e); e.a = "fn_mut".to_string(); };
// ...
```

可以看出 **FnMut** 类型的闭包是可以运行多次的，且可以修改捕获变量的值。

- **Fn**

```rust
#[lang = "fn"]
pub trait Fn<Args>: FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}
```

参数类型是 ```&self```，所以，这种类型的闭包是不可变借用，不会改变变量，也不会释放该变量。所以可以运行多次。

例子:

```rust
fn fn_immut<F>(mut func: F) where F: Fn() {
    func();
    func();
}

// ...
let e = E { a: "fn_once".to_string() };
let f = || { println!("Fn closure calls: {:?}", e); };
fn_immut(f);
// ...
```

可以看出 **Fn** 类型的闭包是可以运行多次的，但不可以修改捕获变量的值。

### 常见的错误

有时候在使用 **Fn/FnMut** 这里类型的闭包，编译器经常会给出这样的错误:

```
# ...
cannot move out of captured variable in an Fn(FnMut) closure
# ...
```

看下如何复现这种情形:

```rust
fn main() {
    fn fn_immut<F>(f: F) where F: Fn() -> String {
        println!("calling Fn closure from fn, {}", f());
    }
    
    let a = "Fn".to_string();
    fn_immut(|| a); // 闭包返回一个字符串
}
```

这样写就会出现上面的那种错误。但如何修复呢？

```rust
fn_immut(|| a.clone());
```

但原因是什么呢？

只要稍稍改下上面的代码，运行一次，编译器给出的错误就很明显了:

```rust
fn main() {
    fn fn_immut<F>(f: F) where F: Fn() -> String {
        println!("calling Fn closure from fn, {}", f());
    }
    
    let a = "Fn".to_string();
    let f = || a;
    fn_immut(f);
}
```

编译器给出的错误如下:

```
7 |     let f = move || a;
  |             ^^^^^^^^-
  |             |       |
  |             |       closure is `FnOnce` because it moves the variable `a` out of its environment
  |             this closure implements `FnOnce`, not `Fn`
8 |     fn_immut(f);
  |     -------- the requirement to implement `Fn` derives from here
```

看到没，编译器推导出这个闭包是 **FnOnce** 类型的，因为闭包最后返回了 **a** ，交还了所有权，是不能再运行第二次了，因为闭包不再是 **a** 的所有者。

而 **Fn/FnMut** 是被认定可以多次运行的，如果交还了捕获变量的所有权，则下次就不能运行了，所以会报出前面那个错误。


### 结语

因为闭包和rust里的生命周期，所有权紧紧联系在一起了，有时候不怎么好理解，但多写写代码，多试几次，大概就能明白这三者之间的区别。

总之，闭包是rust里非常好用的功能，能让代码变得简洁优雅，是值得去学习和掌握的！