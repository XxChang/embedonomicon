# 全局单例

在这部分我们将说到如何实现一个全局的，共享单例。 The embedded Rust book 提到了局部的，拥有所有权的单例，
这对Rust来说几乎是独一无二的。全局单例本质上是你在C和C++中见到的单例模式；它们并不止出现在嵌入式开发中，但
是因为它们涉及到符号，所以似乎很适合嵌入式宝典。

> **TODO**(resources team) link "the embedded Rust book" to the singletons
> section when it's up

为了解释这部分，我们将扩展我们在上部分开发的日志以支持全局日志记录。结果与在the embedded Rust book提到
的`#[global_allocator]`功能非常相似。

> **TODO**(resources team) link `#[global_allocator]` to the collections chapter
> of the book when it's in a more stable location.

总结下我们需要的东西：

在上一部分我们创造了一个`log!`宏以通过一个特定的logger去记录信息，logger是一个实现了`Log` trait的值。
`log!`宏的语法是`log!(logger, "String")`。我们想要扩展下宏，让`log!("String")`也可以工作。
Using the `logger`-less version should log the
message through a global logger; 这是`std::println!`的工作方式。我们也需要一个机制去声明全局logger是什么；这是与`#[global_allocator]`相似的部分。

它可以是在顶层crate中声明的全局logger，它也可以是在顶层crate中定义的全局logger的数据类型
It could be that the global logger is declared in the top crate and it could
also be that the type of the global logger is defined in the top crate. In this
scenario the dependencies can *not* know the exact type of the global logger. To
support this scenario we'll need some indirection.

Instead of hardcoding the type of the global logger in the `log` crate we'll
declare only the *interface* of the global logger in that crate. That is we'll
add a new trait, `GlobalLog`, to the `log` crate. The `log!` macro will also
have to make use of that trait.

``` console
$ cat ../log/src/lib.rs
```

``` rust
{{#include ../ci/singleton/log/src/lib.rs}}
```

There's quite a bit to unpack here.

Let's start with the trait.

``` rust
{{#include ../ci/singleton/log/src/lib.rs:4:6}}
```

`GlobalLog`和`Log`都有一个`log`方法。不同的是，`GlobalLog`将一个共享的引用拿给接收者(`&self`)。
因为全局logger将是一个`static`变量所以这是必须的。之后会提到更多。

另一个不同是，`GlobalLog.log`不返回一个`Result`。 This
means that it can *not* report errors to the caller. This is not a strict
requirement for traits used to implement global singletons. Error handling in
global singletons is fine but then all users of the global version of the `log!`
macro have to agree on the error type. Here we are simplifying the interface a
bit by having the `GlobalLog` implementer deal with the errors.

Yet another difference is that `GlobalLog` requires that the implementer is
`Sync`, that is that it can be shared between threads. This is a requirement for
values placed in `static` variables; their types must implement the `Sync`
trait.

At this point it may not be entirely clear why the interface has to look this
way. The other parts of the crate will make this clearer so keep reading.

Next up is the `log!` macro:

``` rust
{{#include ../ci/singleton/log/src/lib.rs:17:29}}
```

When called without a specific `$logger` the macros uses an `extern` `static`
variable called `LOGGER` to log the message. This variable *is* the global
logger that's defined somewhere else; that's why we use the `extern` block. We
saw this pattern in the [main interface] chapter.

[main interface]: main.html

We need to declare a type for `LOGGER` or the code won't type check. We don't
know the concrete type of `LOGGER` at this point but we know, or rather require,
that it implements the `GlobalLog` trait so we can use a trait object here.

The rest of the macro expansion looks very similar to the expansion of the local
version of the `log!` macro so I won't explain it here as it's explained in the
[previous] chapter.

[previous]: logging.html

Now that we know that `LOGGER` has to be a trait object it's clearer why we
omitted the associated `Error` type in `GlobalLog`. If we had not omitted then
we would have need to pick a type for `Error` in the type signature of `LOGGER`.
This is what I earlier meant by "all users of `log!` would need to agree on the
error type".

Now the final piece: the `global_logger!` macro. It could have been a proc macro
attribute but it's easier to write a `macro_rules!` macro.

``` rust
{{#include ../ci/singleton/log/src/lib.rs:41:47}}
```

This macro creates the `LOGGER` variable that `log!` uses. Because we need a
stable ABI interface we use the `no_mangle` attribute. This way the symbol name
of `LOGGER` will be "LOGGER" which is what the `log!` macro expects.

The other important bit is that the type of this static variable must exactly
match the type used in the expansion of the `log!` macro. If they don't match
Bad Stuff will happen due to ABI mismatch.

Let's write an example that uses this new global logger functionality.

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/singleton/app/src/main.rs}}
```

> **TODO**(resources team) use `cortex_m::Mutex` instead of a `static mut`
> variable when `const fn` is stabilized.

We had to add `cortex-m` to the dependencies.

``` console
$ tail -n5 Cargo.toml
```

``` text
{{#include ../ci/singleton/app/Cargo.toml:11:15}}
```

This is a port of one of the examples written in the [previous] section. The
output is the same as what we got back there.

``` console
$ cargo run | xxd -p
```

``` text
{{#include ../ci/singleton/app/dev.out}}
```

``` console
$ cargo objdump --bin app -- -t | grep '\.log'
```

``` text
{{#include ../ci/singleton/app/dev.objdump}}
```

---

Some readers may be concerned about this implementation of global singletons not
being zero cost because it uses trait objects which involve dynamic dispatch,
that is method calls are performed through a vtable lookup.

However, it appears that LLVM is smart enough to eliminate the dynamic dispatch
when compiling with optimizations / LTO. This can be confirmed by searching for
`LOGGER` in the symbol table.

``` console
$ cargo objdump --bin app --release -- -t | grep LOGGER
```

``` text
{{#include ../ci/singleton/app/release.objdump}}
```

If the `static` is missing that means that there is no vtable and that LLVM was
capable of transforming all the `LOGGER.log` calls into `Logger.log` calls.
