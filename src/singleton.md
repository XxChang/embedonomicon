# 全局单例

在这部分会说到如何实现一个全局的，共享单例。 The embedded Rust book 中提到了局部的，拥有所有权的单例，
这对Rust来说几乎是独一无二的。全局单例本质上就是在C和C++中见到的那些单例模式；它们不止出现在嵌入式开发中，但是因为它们涉及到符号，所以似乎很适合这本书的内容。

> **TODO**(resources team) link "the embedded Rust book" to the singletons
> section when it's up

为了解释这部分，我们将扩展我们在上部分开发的日志以支持全局日志记录。结果与在the embedded Rust book提到的`#[global_allocator]`功能非常相似。

> **TODO**(resources team) link `#[global_allocator]` to the collections chapter
> of the book when it's in a more stable location.

总结下我们需要的东西：

在上一部分我们创造了一个`log!`宏以通过一个特定的logger去记录信息，logger是一个实现了`Log` trait的值。
`log!`宏的语法是`log!(logger, "String")`。我们想要扩展下宏，让`log!("String")`也可以工作。没有`logger`的版本的宏可以通过一个全局的logger去记录信息；这是`std::println!`的工作方式。我们也需要一个机制去声明全局logger是什么；这部分与`#[global_allocator]`相似。

可能是在top crate中声明了全局logger，也可能是在top crate中定义了全局logger的类型。在这种情况下，依赖*不*知道全局logger的确切类型。为了支持这种情况，我们需要一些间接方法。

我们只在`log`库中声明全局logger的*接口*，而不是在`log`库中硬编码全局logger的类型。我们将会给`log`库添加一个新的trait，`GlobalLog`。`log!`宏也必须要使用这个trait 。

``` console
$ cat ../log/src/lib.rs
```

``` rust
{{#include ../ci/singleton/log/src/lib.rs}}
```

这里有很多东西要拆开来看。

先从trait开始。

``` rust
{{#include ../ci/singleton/log/src/lib.rs:4:6}}
```

`GlobalLog`和`Log`都有一个`log`方法。不同的是，`GlobalLog`需要获取一个对接收者的共享的引用(`&self`)。
因为全局logger是一个`static`变量所以这是必须的。之后会提到更多。

另一个不同是，`GlobalLog.log`不返回一个`Result`。 这意味着它*不*会向调用者报告错误。这对用来实现全局单例的traits来说不是一个严格的要求。全局单例中的错误处理很好，但是另一方面全局版本的`log!`宏的的所有用户必须就错误类型达成一致。这里通过让`GlobalLog`的实现者来处理错误可以简化这个接口。

还有另一个不同，`GlobalLog`要求实现者是`Sync`的，它可以在线程间被共享。对于放置在`static`变量中的值来说这是一个要求；它们的类型必须实现`Sync` trait 。

此时可能还不完全清除接口为什么必须要这样。库的其它部分将会解释得更清楚，所以请继续读下去。

接下来是`log!`宏：

``` rust
{{#include ../ci/singleton/log/src/lib.rs:17:29}}
```

当不使用一个指定的`$logger`去调用宏的时候，宏会使用一个被叫做`LOGGER`的`extern` `static`变量去记录信息。这个变量*是*定义在其它地方的全局logger；这就是为什么我们会使用`extern`块。我们可以在[main接口]章节中看到这种模式。

[main接口]: main.html

我们需要声明一个与`LOGGER`有关的类型要不然代码不会做类型检查。此时我们不知道`LOGGER`的具体类型，但是我们知道，或者至少要求，它要实现`GlobalLog` trait，所以这里我们可以使用一个trait对象。

剩余的宏展开与局部版本的`log!`宏展开很像，因此我不会在这里解释它，因为在[先前]的章节中已经解释过了。

[先前]: logging.html

现在我们知道`LOGGER`必须是一个trait对象，为什么我们要在`GlobalLog`中去掉关联的`Error`类型更清楚了。如果我们没有去掉`Error`类型，那么我们将需要为`LOGGER`的类型签名中的`Error`挑选一个类型。这就是我之前提到的“`log!`的所有用户需要对错误类型达成一致”。

现在是最后的片段：`global_logger!`宏。它可以是一个过程宏attribute，但是写一个`macro_rules!`宏更简单。

``` rust
{{#include ../ci/singleton/log/src/lib.rs:41:47}}
```

这个宏生成了`log!`宏要使用的`LOGGER`变量。因为我们需要一个稳定的ABI接口，所以我们使用了`no_mangle` attribute 。这样子的话，`LOGGER`的符号名就会是`LOGGER`，这是`log!`宏期望的符号名。

另外重要的一点是，这个静态变量的类型必须精确地匹配在`log!`宏的展开中所使用的类型。如果它们不匹配，由于ABI的误匹配将会导致坏事发生。

让我们来写一个使用这个新的全局logger功能的例子。

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/singleton/app/src/main.rs}}
```

> **TODO**(resources team) use `cortex_m::Mutex` instead of a `static mut`
> variable when `const fn` is stabilized.

我们必须添加`cortex-m`到这个依赖上。

``` console
$ tail -n5 Cargo.toml
```

``` text
{{#include ../ci/singleton/app/Cargo.toml:11:15}}
```

这是在上个章节中所写的某个例子的移植。这里的输出和之前的是一样的。
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

一些读者可能会担心这种全局单例的实现不是零开销的，因为使用trait对象涉及到动态分发(dynamic dispatch)，通过一个虚表(vtable)查找去执行方法调用。

然而，LLVM足够聪明，可以在使用优化/LTO编译时，消除动态分发。通过在符号表中搜索`LOGGER`可以确认这一点。

``` console
$ cargo objdump --bin app --release -- -t | grep LOGGER
```

``` text
{{#include ../ci/singleton/app/release.objdump}}
```

如果没有`static`，则意味着没有虚表，LLVM能够把所有的`LOGGER.log`调用变成`Logger.log`调用。
