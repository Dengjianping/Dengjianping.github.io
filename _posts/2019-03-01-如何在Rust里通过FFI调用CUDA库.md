---
layout: post
title: "如何在Rust里通过FFI调用CUDA库"
author: Jamie
---

目前Rust对高性能密集运算支持得不太好。虽然现在目前开始支持SIMD了，但也只是初步支持。整个生态还处于早期阶段，官方也没有什么工作组成立。

这块领域基本上是c/c++的天下。目前可以做高性能计算的并行框架的有一些，Nvidia的CUDA，比较普遍的OpenCL(基本上所有平台都支持)，MPI等等。

而我自己之前写过一段时间的CUDA，所以有点经验，CUDA在性能上比OpenCL(我没写过OpenCL，但有测评)要高些，OpenCL的发展速度也远远比不上CUDA，而且调试工具强大。只要学过c/c++，学习成本基本上不大。CUDA代码的调优倒是后期的比较费力的事情，而且因人能力而异，调优出来的性能差别比较大。CUDA的生态也要强大得多，比如各种机器学习的框架支持。

这篇文章就是简单讲下如何rust通过FFI来调用CUDA生成的静态库，来实现高性能计算。

我们需要一个crate来实现FFI：
- [**libc**](https://docs.rs/libc/0.2.48/libc/)

目前Rust对CUDA(编译器也是llvm)支持得不太好，而且是[nightly feature](https://doc.rust-lang.org/unstable-book/language-features/abi-ptx.html)。
值得推荐的crate是arrayfire-rust，不过只是用rust做了wrapper。
- [**arrayfire-rust**](https://github.com/arrayfire/arrayfire-rust)。[**ArrayFire**](https://arrayfire.com/)底层是c/c++写的，平台支持包括OpenCL和CUDA，rust作为wrapper。支持线性代数，信号处理(傅立叶变换等等)，图像处理，机器学习等等。

不过也有一些crate来支持CUDA，但我没具体了解过，有兴趣可以去了解下：
- [**RustaCUDA**](https://github.com/bheisler/RustaCUDA)
- [**rust-ptx-builder**](https://github.com/denzp/rust-ptx-builder)

关于整个示例代码(不会讲实现细节)，可以看这个repo：
- [**minimal_block_chain**](https://github.com/Dengjianping/minimal_block_chain)。这是那篇著名文章关于如何编写一个简单的区块链的文章 [**Learn Blockchains by Building One**](https://hackernoon.com/learn-blockchains-by-building-one-117428612f46?gi=8e1bb887685f)。文章是用python实现的，我希望能用rust来实现一遍。其中后端的挖矿代码使用CUDA实现的。目前工程还处于未完工状态，但CUDA代码部分写了一部分，但编译没问题，所以就用这个来作为示例。

## 详情

### 接口部分
CUDA部分

```
extern "C" const char* SHA256(const char *input) {
    // ...
}
```

Rust部分
```
// link属性里指明链接库的名字，而且类型
#[link(name="cuda_crypto", kind="static")]
extern {
    // 名字要对应于CUDA的接口名字
    pub fn SHA256(a: *const c_char) -> *const c_char;
}

//...

// 可以看到CString到String的转换是比较麻烦的
pub fn cuda_sha256<T: AsRef<str>>(t: T) -> String {
    let c_str = CString::new(t.as_ref()).unwrap();
    // 必须unsafe调用
    let r_ptr = unsafe { SHA256(c_str.as_ptr()) }; // c_char
    let r_str: &CStr = unsafe { CStr::from_ptr(r_ptr) }; // &CStr
    let str_slice = r_str.to_str().unwrap(); // &str
    str_slice.to_string() // String
}

```

### CUDA工程的编译设置
本人不会写makefile，但会写CMake，且CMake目前也支持CUDA。
下面就是整个CMakeLists的内容:
```
# because cmake fully supports cuda until 3.8
cmake_minimum_required(VERSION 3.8)
project(cuda_crypto LANGUAGES CXX CUDA)
add_compile_options(-std=c++11)

set(CMAKE_CUDA_FLAGS "-arch=sm_35 -Xcompiler -fPIC")
set(CMAKE_CUDA_FLAGS_RELEASE "-O3 -DNDEBUG")

add_library(
    cuda_crypto STATIC # generate .a file
    ${PROJECT_SOURCE_DIR}/src/sha.h
    ${PROJECT_SOURCE_DIR}/src/md5.cu
    ${PROJECT_SOURCE_DIR}/src/sha256.cu
    ${PROJECT_SOURCE_DIR}/src/sha384.cu
    ${PROJECT_SOURCE_DIR}/src/sha512.cu
)

# set_target_properties(cuda_crypto PROPERTIES PREFIX "")
install(TARGETS cuda_crypto DESTINATION .)
```
这个文件很简单，就是编译整个工程，生成一个静态文件。
不过有个地方学必须要说明一下，有个flag，```-Xcompiler -fPIC```。如果你是生成静态库给别人调用，这个flag必要要设置。不然就会报如下类似的链接错误。

```
error: linking with `cc` failed: exit code: 1
# 中间一大堆错误
...
...
...
can not be used when making a shared object; recompile with -fPIC
...
...
```

### Rust Types VS C/C++ Types
Rust的类型对应于C/C++的类型。这里我就不细讲了，大家可以看这个文档:
- [Types](https://docs.rs/libc/0.2.49/libc/#types)

### 整个工程的编译

接下来要讲下如何在cargo里编译这个CUDA工程。cargo支持自定义工程编译，只要在cargo.toml文件里添加build = "build.rs"。还有我们需要一个crate来帮助编译cuda工程
- [**cmake**](https://crates.io/crates/cmake)。只要定义好了CmakeLists.txt，就可以正常编译CUDA工程了。

看下部分cargo.toml的build部分。

```
[package]
# ......
build = "build.rs" 

# and this section
[build-dependencies] 
cmake = "0.1"
```

这个是build.rs的内容。

```rust
use cmake;

fn main() {
    // CUDA工程的目录
    let dst = cmake::Config::new("cuda_crypto").build();
    
    // rustc要搜索cuda静态库的目录
    println!("cargo:rustc-link-search=native={}", dst.display());
    // rustc要链接库的名字
    println!("cargo:rustc-link-lib=static=cuda_crypto");
    
    println!("cargo:rustc-link-lib=dylib=stdc++"); // link to stdc++ lib
    
    // 链接CUDA的运行时库
    let lib_path = env!("LD_LIBRARY_PATH");
    // 找到包含CUDA运行时的静态库目录
    let cuda_lib_path: Vec<_> = lib_path.split(':').into_iter().filter(|path| path.contains("cuda")).collect();
    if cuda_lib_path.is_empty() {
        panic!("Ensure cuda installed on your environment");
    } else {
        println!("cargo:rustc-link-search=native={}", cuda_lib_path[0]);
        println!("cargo:rustc-link-lib=cudart"); // cuda run-time lib
    }
}
```

有两点要说明下：
- 因为CUDA代码用到了c++ 11 feature，而且CMake文件里也开启了c++ 11选项，所以需要rustc需要链接c++的标准库。

```
println!("cargo:rustc-link-lib=dylib=stdc++");
```

不然会报出如下类似错误:

```
error: linking with `cc` failed: exit code: 1
# 中间一大堆错误
...
...
 undefined reference to symbol '_ZNKSt7__cxx1118basic_stringstreamIcSt11char_traitsIcESaIcEE3strEv@@GLIBCXX_3.4.21'
/usr/lib/aarch64-linux-gnu/libstdc++.so.6: error adding symbols: DSO missing from command line
collect2: error: ld returned 1 exit status
...
...
```

- 当链接CUDA工程的静态库时，还需要链接CUDA的运行时，所以要rustc还要链接**cudart** 。
```
println!("cargo:rustc-link-lib=cudart");
```

不然会报出如下类似错误:

```
error: linking with `cc` failed: exit code: 1
# 中间一大堆错误
...
...
/home/nvidia/Desktop/rust/block_chain/target/debug/build/block_chain-33a39bf5e1ee0159/out/libcuda_crypto.a(sha512.cu.o): In function `cudaError cudaLaunch<char>(char*)':
/usr/local/cuda-9.0/bin/../targets/aarch64-linux/include/cuda_runtime.h:1879: undefined reference to `cudaLaunch'
collect2: error: ld returned 1 exit status
...
...
```

### 结语
我用Rust来调用CUDA的静态库，进了一些坑，所以把一些过程写下来，以免自己以后再犯错，也希望能够帮助别人绕过这些坑。

如有错误，欢迎指出，谢谢！