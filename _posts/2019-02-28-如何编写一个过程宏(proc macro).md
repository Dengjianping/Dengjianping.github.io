---
layout: post
title: "如何编写一个过程宏(proc-macro)"
author: Jamie
---

过程宏是rust里的强大的武器，非常值得学习rust的人去掌握。但过程宏的编写有点难度，且文档也不太详细，最近也专门学习了下过程宏，算是有点收获，写下一点东西。

## 优点
- 增加代码的复用。
- ~~性能。因为是在编译时生成，所以会得到更好的性能。~~没测试过，有待商榷

## Reference

这里也有一些官方的博文和文档可以帮助理解过程宏，去google里搜也能得到一些有用的文章。

[**官方文档**](https://doc.rust-lang.org/reference/procedural-macros.html)

[**官方的blog**](https://blog.rust-lang.org/2018/12/21/Procedural-Macros-in-Rust-2018.html)

[**Introduction to Procedural Macros in Rust**](https://tinkering.xyz/introduction-to-proc-macros/)

## 过程宏的分类
- **proc-macro**
- **proc-macro-derive**
- **proc-macro-attribute**


## 构建过程宏的必要设置
构建过程宏，要在cargo.toml里面设置一些参数，这是必须的。一般来说，过程宏必须是一个库，或者作为工程的子库，不能单独作为一个源文件存在，至少目前不行。
```
[lib]
proc-macro = true
path = "src/lib.rs"
```

而编写过程宏，在stable版本里，我们需要借助三个crate:

- [**syn**](https://docs.rs/syn/0.15.26/syn/index.html)，这个是用来解析语法树(AST)的。各种语法构成
- [**quote**](https://docs.rs/quote/0.6.11/quote/)，解析语法树，生成rust代码，从而实现你想要的新功能。
- [**proc_macro**(std)](https://doc.rust-lang.org/proc_macro/index.html) 和 [**proc_macro2**(3rd-party)](https://docs.rs/proc-macro2/0.4.27/proc_macro2/index.html)

但在nightly版本里，以上的这些crate都不需要了，不依赖第三方crate，还有就是语法上是稍微有些不同，大部分是一样的。但这篇文章只讲stable rust里的过程宏，如果想了解nightly rust的过程宏，可以去看[**maud**](https://github.com/lfairy/maud) 和[**Rocket**](https://github.com/SergioBenitez/rocket)，前者是一个HTML模板引擎，大量使用了过程宏，模板都是编译时生成，所以性能非常高，而后者是一个web framework，rust各种黑魔法使用的集大成者。

## 例子

### proc-macro(function-like，类函数宏)
这种过程宏和标准宏很类似，只是构建过程不太一样，使用方式还是一样的。标准语法是这样的。
```rust
#[proc_macro]
pub fn my_proc_macro(input: TokenStream) -> TokenStream{
    // ...
}
```
可以看出函数式的过程宏只接受一个形参，而且必须是pub的。
简单写一个例子，参照官网文档的，只是稍微改了一点点。
```rust
#[proc_macro]
pub fn my_proc_macro(ident: TokenStream) -> TokenStream {
    let new_func_name = format!("test_{}", ident.to_string());
    let concated_ident = Ident::new(&new_func_name, Span::call_site()); // 创建新的ident，函数名

    let expanded = quote! {
        // 不能直接这样写trait bound，T: Debug
        // 会报错，找不到Debug trait，最好给出full path
        fn #concated_ident<T: std::fmt::Debug>(t: T) {
            println!("{:?}", t);
        }
    };
    expanded.into()
}
```

使用情形如下。
```rust
use your_crate_name::my_proc_macro;
// ...
my_proc_macro!(hello)!; // 函数test_hello就生成了，可见性在调用之后
// ...
test_hello("hello, proc-macro");
test_hello(10);
```

可以看出，写一个函数式的过程宏还是不那么复杂的。

### proc_macro_derive(Derive mode macros, 继承宏)
继承宏的函数签名和前者有些类似:

```rust
#[proc_macro_derive(MyDerive)]
pub fn my_proc_macro_derive(input: TokenStream) -> TokenStream{
    // ...
}
```

不过不同的是，引入属性有些不同。
```
#[proc_macro_derive(MyDerive)]
```

**proc_macro_derive**表明了这是继承宏，还定义了新的继承宏的名字**MyDerive**。
熟悉rust编程的，都应该知道有个继承宏，一直用得到，就是**Debug**。这是标准库里的，可以帮助调试和显示。所以呢，这里就来实现一个类似功能的继承宏，暂时命名这个过程宏名字为**Show**。
这个例子稍微有点复杂。当然我觉得还是先看了官方文档的例子之后再来看我的例子会比较好些。
```rust
#[proc_macro_derive(Show)]
pub fn derive_show(item: TokenStream) -> TokenStream {
    // 解析整个token tree
    let input = parse_macro_input!(item as DeriveInput);
    let struct_name = &input.ident; // 结构体名字

    // 提取结构体里的字段
    let expanded = match input.data {
        Data::Struct(DataStruct{ref fields,..}) => {
            if let Fields::Named(ref fields_name) = fields {
                // 结构体中可能是多个字段
                let get_selfs: Vec<_> = fields_name.named.iter().map(|field| {
                    let field_name = field.ident.as_ref().unwrap(); // 字段名字
                    quote! {
                        &self.#field_name
                    }
                }).collect();

            let implemented_show = quote! {
                // 下面就是Display trait的定义了
                // use std::fmt; // 不要这样import，因为std::fmt是全局的，无法做到卫生性(hygiene)
                // 编译器会报错重复import fmt当你多次使用Show之后
                impl std::fmt::Display for #struct_name {
                    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
                        // #(#get_self),*，这是多重匹配，生成的样子大概是这样：&self.a, &self.b, &self.c, ...
                        // 用法和标准宏有点像，关于多个匹配，可以看这个文档
                        // https://docs.rs/quote/0.6.11/quote/macro.quote.html
                        write!(f, "{} {:?}", stringify!(#struct_name), (#(#get_selfs),*))
                    }
                }
            };
            implemented_show
            
            } else {
                panic!("sorry, may it's a complicated struct.");
            }
        }
        _ => panic!("sorry, Show is not implemented for union or enum type.")
    };
    expanded.into()
}
```

使用情形:
```rust
use your_crate_name::Show;
// ...
#[derive(Show)]
struct MySelf {
    name: String,
    age: u8,
}
// ...
let me = MySelf{name: "Jamie", age: 255};
println!("{}", me); // MySelf (Jamie, 255)
```

不过呢，继承宏还可以添加额外的属性，函数签名类似如下
```rust
#[proc_macro_derive(MyDerive, attributes(my_attr)]
pub fn my_proc_macro_derive(input: TokenStream) -> TokenStream{
    // ...
}
```

这里增加了一个关键字**attributes**，并指定了属性的名字。详细情况可以看官方文档。示例代码里也有个例子，因为文章篇幅，我就不赘述了。

### proc_macro_attribute(Attribute macros, 属性宏)
属性宏的函数签名类似如下:
```rust
#[proc_macro_attribute]
pub fn my_attribute_macro(attr: TokenStream, item: TokenStream) -> TokenStream {
    // ...
}
```

可以看到这里的形参是两个，使用的关键字是**proc_macro_attribute**。
关于例子，熟悉python的人应该知道修饰器吧，其实本质就是函数(闭包)可以作为一个对象来返回。
比如我需要一个修饰器来测量一个调用函数的运行时间。python的实现很简单，如下：
```python
def my_decorator(func):
    import time
    def timming_measrement(*args):
        start = time.time()
        func(*args)
        end = time.time()
        print(f"time cost: {end - start}")
    return timming_measrement
    
@my_decorator
def my_target_func(sec):
    import time
    time.sleep(sec)
    
my_target_func(2) # should print 2.00xx
my_target_func(4) # should print 4.00xx
```

如果要用rust来实现类似功能的代码，就要复杂一些了。
属性宏接受的参数也不太一样，这也会导致属性宏的实现也会不太一样：
```rust
// 可能属性参数多种多样
// #[my_macro_attribute]
// #[my_macro_attribute=something]
#[my_macro_attribute(post)] // 这是例子的使用情况
fn my_func() {
    // ...
}
```

实现过程

```rust
#[proc_macro_attribute]
pub fn rust_decorator(attr: TokenStream, func: TokenStream) -> TokenStream {
    let func = parse_macro_input!(func as ItemFn); // 我们传入的是一个函数，所以要用到ItemFn
    let func_name = &func.ident; // 函数名
    let func_vis = &func.vis; // pub
    let func_decl = &func.decl; // 函数申明
    let func_block = &func.block; // 函数主体实现部分{}

    let func_generics = &func_decl.generics; // 函数泛型
    let func_inputs = &func_decl.inputs; // 函数输入参数
    let func_output = &func_decl.output; // 函数返回

    // 提取参数，参数可能是多个
    let params: Vec<_> = func_inputs.iter().map(|i| {
        match i {
            // 提取形参的pattern
            // https://docs.rs/syn/0.15.26/syn/enum.Pat.html
            FnArg::Captured(ref val) => &val.pat, // pat没有办法移出val，只能借用，或者val.pat.clone()
            _ => unreachable!("it's not gonna happen."),
        }
    }).collect();
    
    // 解析attr
    let attr = parse_macro_input!(attr as AttributeArgs);
    // 提取attr的ident，此处例子只有一个attribute
    let attr_ident = match attr.get(0).as_ref().unwrap() {
        NestedMeta::Meta(Meta::Word(ref attr_ident)) => attr_ident.clone(),
        _ => unreachable!("it not gonna happen."),
    };
    
    // 创建新的ident, 例子里这个ident的名字是time_measure
    // let attr = Ident::new(&attr.to_string(), Span::call_site());
    let expanded = quote! { // 重新构建函数执行
        #func_vis fn #func_name #func_generics(#func_inputs) #func_output {
            // 这是没有重新构建的函数，最开始声明的，需要将其重建出来作为参数传入，
            // fn time_measure<F>(func: F) -> impl Fn(u64) where F: Fn(u64)
            // fn deco(t: u64) {
            //     let secs = Duration::from_secs(t);
            //     thread::sleep(secs);
            // }
            fn rebuild_func #func_generics(#func_inputs) #func_output #func_block
            // 注意这个#attr的函数签名：fn time_measure<F>(func: F) -> impl Fn(u64) where F: Fn(u64)
            // 形参是一个函数，就是rebuild_func
            let f = #attr_ident(rebuild_func);

            // 要修饰函数的参数，有可能是多个参数，所以这样匹配 #(#params,) *
            f(#(#params,) *)
        }
    };
    expanded.into()
}
```

还有一段代码，这个函数相当于过程宏的属性(参数attr)。
```rust
// use std::time;
// 该函数接受一个函数作为参数，并返回一个闭包，代码很简单，就不解释了。
// thanks for impl trait
fn runtime_measurement<F>(func: F) -> impl Fn(u64) where F: Fn(u64) {
    move |s| {
        let start = time::Instant::now();
        func(s);
        println!("time cost {:?}", start.elapsed());
    }
}
```

假定这是我们要修饰的目标函数。
```rust
#[rust_decorator(runtime_measurement)]
fn deco(t: u64) {
    let secs = Duration::from_secs(t);
    thread::sleep(secs);
}

// ...
deco(4);
deco(2);
```


## 调试
[**quote**](https://docs.rs/quote/0.6.11/quote/)的作者实现了一个[**cargo-expand**]()，专门用来调试过程宏的，可以在编译时展开你定义的过程宏，但我没具体用过。
具体使用可以看这个文档
- [**Doc**](https://github.com/dtolnay/cargo-expand)
- [**Debugging Rust's new Custom Derive system**](https://quodlibetor.github.io/posts/debugging-rusts-new-custom-derive-system/)

我自己来讲，用的都是些比较笨的办法，比如**println!**，**panic!**。
不过在rust 1.32之后，引入了一个非常总要的功能，用于调试代码的宏**dbg!**。
这个宏能非常漂亮地打印你的调试代码。使用也很简单，这里就不展开了。具体使用可以看文档
- [**dbg!**](https://doc.rust-lang.org/std/macro.dbg.html)

不过还是推荐大家使用cargo-expand。

## 结语
过程宏确实是rust里的黑魔法，希望这篇文章能帮助到一些人了解并使用过程宏，体会到rust的强大。当然了，因为我的知识的不足，一些地方可能会显得不够专业和透彻，希望大家能指出。

要学好过程宏，还是要去好好看文档，多多练习，简单到复杂。

所以的实例代码，可以在这里看到，所以的例子都是在rust版本1.32之下编写并通过编译的，最好使用最新的stable rust。当然nightly rust应该也可以编译过。
- [**proc-macro example**](https://github.com/Dengjianping/proc-macro-examples)
