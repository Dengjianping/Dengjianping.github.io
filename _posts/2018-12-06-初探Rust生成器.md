---
layout: post
title: "初探Rust生成器"
author: Jamie
---

这篇文章主要是讲下Rust里面的generator, 虽然这还是个nightly的feature，如果熟悉Python，应该都知道Python里面的generator，generator的用处主要是能节省内存。
比如如果你有很大的一个规律性的数组，你就可以用generator，而不是数组或者列表。

先讲下Python如何是如何得到一个生成器的。
不要用[]，这是得到了一个列表。
```python
list_array = [x for x in range(100)]
```

使用()可以得到一个生成器。
```python
generator_array = (x for x in range(100))
```

当然也可以用函数来生成，需要使用到关键字**yield**。
```
def generator_func():
    for i in range(100):
        yield i
```

比如我们在Rust的示例中要用到的斐波拉契数列的Python实现。
```python
def fib():
    former, latter = 0, 1
    for i in range(100):
        yield latter
        former, latter = latter, former + latter
```

熟悉了Python的生成器，那回到Rust的生成器。
这个例子就拿斐波拉契数列说事吧。
因为还是nightly的feature，所以必须得nightly的rust。
```rust
#![feature(generators, generator_trait)]

use std::ops::{Generator, GeneratorState};

fn rust_generator() -> impl Generator<Yield=u32, ()> {
    move || {
        let (mut former, mut latter): (u32, u32) = (0, 1);
        for i in 0..100 {
            yield latter;
            let (former, latter) = (latter, former + latter);
        }
    }
}

fn main() {
    //let mut g = rust_generator();
    while let GeneratorState::Yielded(n)= unsafe { rust_generator().resume() } {
        println!("{}", n);
    }
}
```

其实每次yield就会返回控制权到main函数，这就是简单的协程。
那如何生成一个Future呢？标准库里提供了一个转换接口。当然这里不是讲标准库将要到来的协程。
```
use std::future::{ Future, from_generator };
```