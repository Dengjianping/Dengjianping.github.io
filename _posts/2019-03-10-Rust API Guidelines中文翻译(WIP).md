---
layout: post
title: "Rust API Guidelines中文翻译(WIP)"
author: Jamie
---

一直以为，在一个team或者一个公司里，代码规范是一件很重要的事情，大家都遵循一个规则，这既能减少不必要的沟通，节省时间，养成个人的良好习惯等等。如果是一个开源项目，良好的代码规范和API设计，直白且易用，减少开发者的熟悉时间等等。

而由于Rust语言的特殊性，比如所有权，trait等的一些特性，所以很有必要有一套Rust的编程规范，所以官方就编写了一些文档关于规范Rust编程的。

- [**Rust API Guidelines**](https://rust-lang-nursery.github.io/api-guidelines/)

但由于是英文的，所以看起来有时候并不直白，我就花时间翻译了中文版。英文水平有限，翻译得一般，有错误尽管提出。

目前还处在翻译中。

### Rust API Guidelines的章节

#### 关于(About)

- [关于]()
- [About](https://rust-lang-nursery.github.io/api-guidelines/about.html)

#### 清单(Checklist)

- [清单]()
- [Checklist](https://rust-lang-nursery.github.io/api-guidelines/checklist.html)

#### 命名规范(Naming)

- [命名规范]() - Done
- [Naming](https://rust-lang-nursery.github.io/api-guidelines/naming.html)

#### 互操作性规范(Interoperability)

- [互操作性规范]() - Ongoing
- [Interoperability](https://rust-lang-nursery.github.io/api-guidelines/interoperability.html)

#### 宏规范(Macros)

- [宏规范]() - Ongoing
- [Macros](https://rust-lang-nursery.github.io/api-guidelines/macros.html)

#### 文档(Documentation)

- [文档规范]() - Ongoing
- [Documentation](https://rust-lang-nursery.github.io/api-guidelines/documentation.html)
 
#### 可预见性规范(Predictability)

- [可预见性规范]() - Ongoing
- [Predictability](https://rust-lang-nursery.github.io/api-guidelines/predictability.html)

#### 灵活性规范(Flexibility)

- [灵活性规范]() - Ongoing
- [Flexibility](https://rust-lang-nursery.github.io/api-guidelines/flexibility.html)

#### 类型安全规范(Type safety)

- [类型安全规范]() - Ongoing
- [Type safety](https://rust-lang-nursery.github.io/api-guidelines/type-safety.html)

#### 依赖性规范(Dependability)

- [依赖性规范]() - Ongoing
- [Dependability](https://rust-lang-nursery.github.io/api-guidelines/dependability.html)

#### 调试性规范(Debuggability)

- [调试性规范]() - Done
- [Debuggability](https://rust-lang-nursery.github.io/api-guidelines/debuggability.html)

#### 防护性规范(Future proofing)

- [防护性规范]() - Ongoing
- [Future proofing](https://rust-lang-nursery.github.io/api-guidelines/future-proofing.html)

#### 必要性规范(Necessities)

- [必要性规范]() - Ongoing
- [Necessities](https://rust-lang-nursery.github.io/api-guidelines/necessities.html)

#### 外部链接

- [RFC 199](https://github.com/rust-lang/rfcs/blob/master/text/0199-ownership-variants.md) - 所有权命名规范
- [RFC 344](https://github.com/rust-lang/rfcs/blob/master/text/0344-conventions-galore.md) - 命名规范
- [RFC 430](https://github.com/rust-lang/rfcs/blob/master/text/0430-finalizing-naming-conventions.md) - 命名规范
- [RFC 505](https://github.com/rust-lang/rfcs/blob/master/text/0505-api-comment-conventions.md) - 文档规范
- [RFC 1574](https://github.com/rust-lang/rfcs/blob/master/text/1574-more-api-documentation-conventions.md) - 文档规范
- [RFC 1687](https://github.com/rust-lang/rfcs/pull/1687) - crate级别文档规范
- [Elegant Library APIs in Rust](https://deterministic.space/elegant-apis-in-rust.html) - 优雅的库API设计，很值得一读，虽然写了已有两年多了
- [Rust Design Patterns](https://github.com/rust-unofficial/patterns) - Rust的设计模式
