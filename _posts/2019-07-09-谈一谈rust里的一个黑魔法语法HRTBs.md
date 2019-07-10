---
layout: post
title: "谈一谈rust里的一个黑魔法语法HRTBs"
author: Jamie
---

最近在reddit的rust板块，看到一则帖子，讲rust expert意味着什么。

- [what_does_it_mean_to_be_an_expert_in_rust](https://www.reddit.com/r/rust/comments/c51uge/what_does_it_mean_to_be_an_expert_in_rust/)

里面就有一个回复说是理解使用HRTBs。当然了，我肯定不是什么expert。所以顺便去了解下什么是HRTBs，于是有了这篇文章。

HRTBs，aka Higher-Ranked Trait Bounds。定义如下
```rust
for<'a> T: Trait<'a>
```

看到定义，这个肯定和生命周期有关系，所以要理解这个，基本的生命周期知识还是需要的。



这里列举了一些理解HRTBs的文档和引用：

- [rfc](https://github.com/rust-lang/rfcs/blob/master/text/0387-higher-ranked-trait-bounds.md). 讲述当年为什么会有HRTBs。
- [nomicon](https://doc.rust-lang.org/nightly/nomicon/hrtb.html)
- [std](https://doc.rust-lang.org/reference/trait-bounds.html#higher-ranked-trait-bounds)
- [stackoverflow](https://stackoverflow.com/questions/35592750/how-does-for-syntax-differ-from-a-regular-lifetime-bound). 觉得我啰嗦没讲清楚，推荐可以看这个。

为什么会出现HRTBs？官方给出的解释，引入HRTBs大部分原因是因为闭包。

#### 例子

先拿出官方的例子(来自[std](https://doc.rust-lang.org/reference/trait-bounds.html#higher-ranked-trait-bounds)，例子本身没多大意义，只是为了说明这个问题)

```rust
fn call_on_ref_zero<F>(f: F) where for<'a> F: Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
```

这里调用函数接受一个闭包作为参数，闭包显示地标明了参数的生命周期，所以这个编译不会有任何问题。

其实我们知道，像函数参数接受单个引用，是可以省略生命周期的，因为编译器可以自动推导出来，这个时候可以省略for<'a>。如下也是可以编译的(现在在Rust 1.36编译没有问题，但之前的版本的编译器我不知道能不能推导)：

```rust
fn call_on_ref_zero<F>(f: F) where F: Fn(&i32) {
    let zero = 0;
    f(&zero);
}
```

但是如果我稍加修改，闭包接受两个引用，并返回其中一个引用，会怎么样？

```rust
fn call_on_ref_zero<F>(f: F) where F: Fn(&i32, &i32) -> &i32 {
    let zero = 0;
    f(&zero, &zero);
}
```

这个时候编译器就报错了，因为闭包不知道你返回的是哪一个引用，编译器是无法推导返回引用的生命周期，所以试着添加生命周期看看。

```rust
fn call_on_ref_zero<'a, F>(f: F) where F: Fn(&'a i32, &'a i32) -> &'a i32 {
    let zero = 0;
    f(&zero, &zero);
}
```

修改后编译器还是会报错，因为 ```&zero``` 的生命周期是短于 ```'a``` 的，这就是为什么需要引入HRTBs的原因了。

```rust
fn call_on_ref_zero<F>(f: F) where for<'a> F: Fn(&'a i32, &'a i32) -> &'a i32 {
    let zero = 0;
    f(&zero, &zero);
}
```

修改成这样，编译就完全没有问题了。这个时候 ```&zero``` 的生命周期可以不短于 ```'a``` 了。


#### 结语

总的来讲，个人觉得，这个本质上还是生命周期的问题，某种程度上，增加了学习生命周期的难度，但也丰富了生命周期的表达力。因为这个语法在Rust里面大部分都是用于闭包场景。对于急于学习rust语言语法的同学来说，这个不那么紧急，除非你很喜欢函数式编程，那么掌握这个是有必要的。

最后给出一个reddit上终极烧脑的 HRTBs 例子，看看能打印什么结果(莫被吓到了)。

```rust
impl<'four> For for &'four for<'fore> For where for<'fore> For: For,
{
    fn four(self: &&'four for<'fore> For) {
        print!("four")
    }
}

fn main() {
    four(&(four as for<'four> fn(&'four for<'fore> For)))
}

trait For {
    fn four(&self) {}
}

fn four(four: &for<'four> For) {
    <&for<'four> For as For>::four(&{
        ((&four).four(), four.four());
        four
    })
}

impl For for for<'four> fn(&'four for<'fore> For) {
    fn four(&self) {
        print!("for")
    }
}
```