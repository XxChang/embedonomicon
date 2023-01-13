# 最小的 `#![no_std]` 程序

在这部分，我们将写一个可以*编译*的最小的 `#![no_std]` 程序。

## `#![no_std]` 是什么意思?

`#![no_std]` 是一个crate层级的属性，其指出crate将链接到 [`core`] 而不是 [`std`] crate，但是这对应用来说意味着什么呢?

[`core`]: https://doc.rust-lang.org/core/
[`std`]: https://doc.rust-lang.org/std/

`std` crate 是Rust的标准库。它包含的功能需要假设程序将运行在一个操作系统上而不是[*直接运行在裸机*]上。`std` 也假设操作系统是一个通用的操作系统，像是会在服务器和桌面看到的那些系统。这样，`std` 在那些经常能在操作系统中遇到的功能：线程，文件，套接字，一个文件系统，进程，等等之上，提供了一个标准的API 。

[*直接运行在裸机上*]: https://en.wikipedia.org/wiki/Bare_machine

换句话说，`core` crate是`std` crate的一个子集，其不对将运行程序的系统做任何假设。它提供与语言的基本类型，像是浮点数，字符串和切片有关的APIs，也提供像是原子操作和SIMD指令这样的与处理器相关的APIs 。然而它缺少涉及到堆内存分配和I/O有关的APIs。

对于一个应用来说，`std` 不仅仅只是提供一种方法访问OS抽象。`std` 在某些情况下，也提供栈溢出保护，处理命令行参数，在一个程序的`main`函数被启动前打开主线程。一个 `#![no_std]` 应用缺少上述的所有运行时，因此它必须在需要的时候初始化它自己的运行时。

由于这些特点，一个 `#![no_std]` 应用可以成为第一个或者是唯一一个运行在一个系统上的代码。它可以成为许多一个标准Rust应用无法成为的东西，比如:

- 一个操作系统的内核。
- 固件。
- 一个启动引导。

## 代码

讲完了，我们可以转向最小的 `#![no_std]` 程序了:

``` console
$ cargo new --edition 2018 --bin app

$ cd app
```

``` console
$ # 把 main.rs 改成这些内容
$ cat src/main.rs
```

``` rust
{{#include ../ci/smallest-no-std/src/main.rs}}
```

这个程序含有一些在标准的Rust程序中不会出现的东西:

`#![no_std]` 属性，我们已经讲过了。

`#![no_main]` 属性，它意味着程序将不会使用标准的 `main` 函数作为入口。在写这本书的时候，Rust的 `main` 接口对程序执行的环境做了一些假设: 比如，它假设存在命令行参数，因此，通常它不适合 `#![no_std]` 程序。 

`#[panic_handler]` 属性。用这个属性标记的函数定义了恐慌的行为，包括库层级的恐慌(`core::painc!`)和语言层级的恐慌(越界索引)。

这个程序不产生任何有用的东西。事实上，它将产生一个空的二进制项。

``` console
$ # 等于 `size target/thumbv7m-none-eabi/debug/app`
$ cargo size --target thumbv7m-none-eabi --bin app
```

``` text
{{#include ../ci/smallest-no-std/app.size}}
```

在链接之前，crate 包含恐慌函数的符号。

``` console
$ cargo rustc --target thumbv7m-none-eabi -- --emit=obj

$ cargo nm -- target/thumbv7m-none-eabi/debug/deps/app-*.o | grep '[0-9]* [^N] '
```

``` text
{{#include ../ci/smallest-no-std/app.o.nm}}
```

然而，它是我们的起点。在下一个部分，我们将搭建一些有用的东西。但是继续之前，让我们设置一个默认的编译目标避免每次调用Cargo不得不传递`--target`标志。

``` console
$ mkdir .cargo

$ # 把 .cargo/config 改成这些内容
$ cat .cargo/config
```

``` toml
{{#include ../ci/smallest-no-std/.cargo/config}}
```

## eh_personality

如果你配置的不是在恐慌时无条件终止(译者注：`panic = "abort"`)，大多数的有完整的操作系统的目标平台都不是(或者如果你的 [自制目标平台][custom-target] 不包含
`"panic-strategy": "abort"`)，那么你必须告诉Cargo这么做或者添加一个 `eh_personality` 函数，后者需要nightly版的编译器。[这里是关于它的Rust文档][more-about-lang-items]，
[这里是一些关于它的讨论][til-why-eh-personality].

在 Cargo.toml 中， 添加:

``` toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

或者，声明 `eh_personality` 函数。一个简单的没有什么特别的东西，展开时和下面一样:

``` rust
#![feature(lang_items)]

#[lang = "eh_personality"]
extern "C" fn eh_personality() {}
```

如果没有这么做，你将会收到这个错误 `language item required, but not found: 'eh_personality'` 。

[custom-target]: ./custom-target.md
[more-about-lang-items]:
  https://doc.rust-lang.org/unstable-book/language-features/lang-items.html#more-about-the-language-items
[til-why-eh-personality]:
  https://www.reddit.com/r/rust/comments/estvau/til_why_the_eh_personality_language_item_is/
