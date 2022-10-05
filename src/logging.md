# 使用符号做日志

这部分将向你展示如何使用符号和ELF格式实现超级廉价的日志。

## 任意符号

每当我们在crates之间需要一个稳定的符号接口时，我们主要使用`no_mangle`属性，有时是`export_name`属性。`export_name`属性采用一个字符串作为符号的名字，而`#[no_mangle]`本质上是`#[export_name = <item-name>]`的语法糖。

事实证明，我们并不局限于单个单词的名字；我们可以使用任意的字符串，比如 语句，作为`export_name`属性的参数。至少当输出格式是ELF时，任何不包含空字节的内容都可以。

让我们查看下:

``` console
$ cargo new --lib foo

$ cat foo/src/lib.rs
```

``` rust
#[export_name = "Hello, world!"]
#[used]
static A: u8 = 0;

#[export_name = "こんにちは"]
#[used]
static B: u8 = 0;
```

``` console
$ ( cd foo && cargo nm --lib )
foo-d26a39c34b4e80ce.3lnzqy0jbpxj4pld.rcgu.o:
0000000000000000 r Hello, world!
0000000000000000 V __rustc_debug_gdb_scripts_section__
0000000000000000 r こんにちは
```

你能看出这有什么用吗?

## 编码

下面是我们要做的：为每个日志信息我们将创造一个`static`变量，但是不是将信息存储*进*变量中，我们将把信息存储进变量的*符号名*中。然后，我们将记录的不是`static`变量的内容，而是它们的地址。

只要`static`变量的大小不是零，每个变量就都会有个不同的地址。这里我们要做的是将每个信息有效地编码为一个唯一的标识符，其恰好是变量的地址。日志系统必须有部分将这个id解码回日志信息。

让我们来编写一些代码解释下这个想法。

在这个例子里我们将需要一些方法来完成I/O操作，因此我们将使用[`cortex-m-semihosting`] crate 。Semihosting是一个技术，其用来让一个目标设备可以借用主机的I/O功能；这里，主机通常是指用来调试目标设备的机器。在我们的例子里，QEMU支持开箱即用的semihosting，因此不需要一个调试器。在一个真正的设备上你可以使用其它方法来进行I/O操作比如一个串口；在这个例子里我们使用semihosting是因为这是在QEMU上进行I/O操作最简单的方法。

[`cortex-m-semihosting`]: https://crates.io/crates/cortex-m-semihosting

这里是代码

``` rust
{{#include ../ci/logging/app/src/main.rs}}
```

我们也可以使用`debug::exit` API来让程序终结QEMU进程。这很方便，因此我们不必手动终结QEMU进程。

这里是Cargo.toml的`dependencies`部分:

``` toml
{{#include ../ci/logging/app/Cargo.toml:7:9}}
```

现在我们可以编译程序了

``` console
$ cargo build
```

为了运行它我们将不得不给我们的QEMU命令添加 `--semihosting-config` 标志:

``` console
$ qemu-system-arm \
      -cpu cortex-m3 \
      -machine lm3s6965evb \
      -nographic \
      -semihosting-config enable=on,target=native \
      -kernel target/thumbv7m-none-eabi/debug/app
```

``` text
{{#include ../ci/logging/app/dev.out}}
```

> **NOTE**: These addresses may not be the ones you get locally because
> addresses of `static` variable are not guaranteed to remain the same when the
> toolchain is changed (e.g. optimizations may have improved).

Now we have two addresses printed to the console.

## Decoding

How do we convert these addresses into strings? The answer is in the symbol
table of the ELF file.

``` console
$ cargo objdump --bin app -- -t | grep '\.rodata\s*0*1\b'
```

``` text
{{#include ../ci/logging/app/dev.objdump}}
$ # first column is the symbol address; last column is the symbol name
```

`objdump -t` prints the symbol table. This table contains *all* the symbols but
we are only looking for the ones in the `.rodata` section and whose size is one
byte (our variables have type `u8`).

It's important to note that the address of the symbols will likely change when
optimizing the program. Let's check that.

> **PROTIP** You can set `target.thumbv7m-none-eabi.runner` to the long QEMU
> command from before (`qemu-system-arm -cpu (..) -kernel`) in the Cargo
> configuration file (`.cargo/conifg`) to have `cargo run` use that *runner* to
> execute the output binary.

``` console
$ head -n2 .cargo/config
```

``` toml
{{#include ../ci/logging/app/.cargo/config:1:2}}
```

``` console
$ cargo run --release
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/app`
```

``` text
{{#include ../ci/logging/app/release.out}}
```

``` console
$ cargo objdump --bin app --release -- -t | grep '\.rodata\s*0*1\b'
```

``` text
{{#include ../ci/logging/app/release.objdump}}
```

So make sure to always look for the strings in the ELF file you executed.

Of course, the process of looking up the strings in the ELF file can be automated
using a tool that parses the symbol table (`.symtab` section) contained in the
ELF file. Implementing such tool is out of scope for this book and it's left as
an exercise for the reader.

## Making it zero cost

Can we do better? Yes, we can!

The current implementation places the `static` variables in `.rodata`, which
means they occupy size in Flash even though we never use their contents. Using a
little bit of linker script magic we can make them occupy *zero* space in Flash.

``` console
$ cat log.x
```

``` text
{{#include ../ci/logging/app2/log.x}}
```

We'll place the `static` variables in this new output `.log` section. This
linker script will collect all the symbols in the `.log` sections of input
object files and put them in an output `.log` section. We have seen this pattern
in the [存储布局] chapter.

[存储布局]: memory-layout.html

The new bit here is the `(INFO)` part; this tells the linker that this section
is a non-allocatable section. Non-allocatable sections are kept in the ELF
binary as metadata but they are not loaded onto the target device.

We also specified the start address of this output section: the `0` in `.log 0
(INFO)`.

The other improvement we can do is switch from formatted I/O (`fmt::Write`) to
binary I/O, that is send the addresses to the host as bytes rather than as
strings.

Binary serialization can be hard but we'll keep things super simple by
serializing each address as a single byte. With this approach we don't have to
worry about endianness or framing. The downside of this format is that a single
byte can only represent up to 256 different addresses.

Let's make those changes:

``` rust
{{#include ../ci/logging/app2/src/main.rs}}
```

Before you run this you'll have to append `-Tlog.x` to the arguments passed to
the linker. That can be done in the Cargo configuration file.

``` console
$ cat .cargo/config
```

``` toml
{{#include ../ci/logging/app2/.cargo/config}}
```

Now you can run it! Since the output now has a binary format we'll pipe it
through the `xxd` command to reformat it as a hexadecimal string.

``` console
$ cargo run | xxd -p
```

``` text
{{#include ../ci/logging/app2/dev.out}}
```

The addresses are `0x00` and `0x01`. Let's now look at the symbol table.

``` console
$ cargo objdump --bin app -- -t | grep '\.log'
```

``` text
{{#include ../ci/logging/app2/dev.objdump}}
```

There are our strings. You'll notice that their addresses now start at zero;
this is because we set a start address for the output `.log` section.

Each variable is 1 byte in size because we are using `u8` as their type. If we
used something like `u16` then all address would be even and we would not be
able to efficiently use all the address space (`0...255`).

## Packaging it up

You've noticed that the steps to log a string are always the same so we can
refactor them into a macro that lives in its own crate. Also, we can make the
logging library more reusable by abstracting the I/O part behind a trait.

``` console
$ cargo new --lib log

$ cat log/src/lib.rs
```

``` rust
{{#include ../ci/logging/log/src/lib.rs}}
```

Given that this library depends on the `.log` section it should be its
responsibility to provide the `log.x` linker script so let's make that happen.

``` console
$ mv log.x ../log/
```

``` console
$ cat ../log/build.rs
```

``` rust
{{#include ../ci/logging/log/build.rs}}
```

Now we can refactor our application to use the `log!` macro:

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/logging/app3/src/main.rs}}
```

Don't forget to update the `Cargo.toml` file to depend on the new `log` crate.

``` console
$ tail -n4 Cargo.toml
```

``` toml
{{#include ../ci/logging/app3/Cargo.toml:7:10}}
```

``` console
$ cargo run | xxd -p
```

``` text
{{#include ../ci/logging/app3/dev.out}}
```

``` console
$ cargo objdump --bin app -- -t | grep '\.log'
```

``` text
{{#include ../ci/logging/app3/dev.objdump}}
```

与之前一样的输出!

## Bonus: Multiple log levels

Many logging frameworks provide ways to log messages at different *log levels*.
These log levels convey the severity of the message: "this is an error", "this
is just a warning", etc. These log levels can be used to filter out unimportant
messages when searching for e.g. error messages.

We can extend our logging library to support log levels without increasing its
footprint. Here's how we'll do that:

We have a flat address space for the messages: from `0` to `255` (inclusive). To
keep things simple let's say we only want to differentiate between error
messages and warning messages. We can place all the error messages at the
beginning of the address space, and all the warning messages *after* the error
messages. If the decoder knows the address of the first warning message then it
can classify the messages. This idea can be extended to support more than two
log levels.

Let's test the idea by replacing the `log` macro with two new macros: `error!`
and `warn!`.

``` console
$ cat ../log/src/lib.rs
```

``` rust
{{#include ../ci/logging/log2/src/lib.rs}}
```

We distinguish errors from warnings by placing the messages in different link
sections.

The next thing we have to do is update the linker script to place error messages
before the warning messages.

``` console
$ cat ../log/log.x
```

``` text
{{#include ../ci/logging/log2/log.x}}
```

We also give a name, `__log_warning_start__`, to the boundary between the errors
and the warnings. The address of this symbol will be the address of the first
warning message.

We can now update the application to make use of these new macros.

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/logging/app4/src/main.rs}}
```

The output won't change much:

``` console
$ cargo run | xxd -p
```

``` text
{{#include ../ci/logging/app4/dev.out}}
```

We still get two bytes in the output but the error is given the address 0 and
the warning is given the address 1 even though the warning was logged first.

Now look at the symbol table.

```  console
$ cargo objdump --bin app -- -t | grep '\.log'
```

``` text
{{#include ../ci/logging/app4/dev.objdump}}
```

There's now an extra symbol, `__log_warning_start__`, in the `.log` section.
The address of this symbol is the address of the first warning message.
Symbols with addresses lower than this value are errors, and the rest of symbols
are warnings.

With an appropriate decoder you could get the following human readable output
from all this information:

``` text
WARNING Hello, world!
ERROR Goodbye
```

---

If you liked this section check out the [`stlog`] logging framework which is a
complete implementation of this idea.

[`stlog`]: https://crates.io/crates/stlog
