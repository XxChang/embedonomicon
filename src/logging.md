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

> **注意**：这些地址可能不是你本地获得的地址因为当工具链被改变时`static`变量的地址不保障仍然是一样的(比如 优化可能会改进地址)。

现在我们有两个地址被打印到控制台上了。

## 解码

我们如何把这些地址转换成字符串呢?答案在ELF文件的符号表中。

``` console
$ cargo objdump --bin app -- -t | grep '\.rodata\s*0*1\b'
```

``` text
{{#include ../ci/logging/app/dev.objdump}}
$ # 第一列是符号地址；最后一列是符号名
```

`objdump -t` 打印符号表。这个表包含*所有*的符号但是我们只查找`.rodata`中的符号，且它的大小只有一个字节(我们的变量类型是`u8`) 。

需要注意的是，在优化程序时，符号的地址将可能改变。让我们检查下。

> **关键技巧** 你可以在Cargo配置文件(`.cargo/config`)中把 `target.thumbv7m-none-eabi.runner` 设置成之前(`qemu-system-arm -cpu (..)`) 的长QEMU命令以让`cargo run`使用这个*runner*执行输出的二进制项。

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

因此，请确保始终在执行的ELF文件中查找字符串。

当然，可以使用一个工具自动检查ELF文件中的字符串，该工具可以解析ELF文件包含的符号表(`.symtab`) 。实现这样的工具超出了这本书的范围，且这是留给读者一个练习。

## 使它变成零开销

我们能做得更好吗?是的，我们可以！

现在的实现把`static`变量放在`.rodata`中，其意味着它们会占用Flash的大小即使我们从不会使用它们的内容。使用一点链接器脚本魔法我们可以让它们占用的Flash空间为*零*。

``` console
$ cat log.x
```

``` text
{{#include ../ci/logging/app2/log.x}}
```

我们将把`static`变量放在这个新的输出`.log` section中。这个链接器脚本将把输入目标文件的`.log` section中的所有符号集中起来并把它们放进一个输出的`.log` section中。我们在[存储布局]章节中已经见过这个方法了。

[存储布局]: memory-layout.html

这块`(INFO)`部分是新的；这告诉链接器这个section是一个非可分配部分。非可分配部分在ELF中作为元数据保留起来但是它们不会被加载到目标设备上。

我们也指定这个输出section的起始地址：`.log 0 (INFO)` 中的`0` 。

我们可以做的其它改进是从格式化的I/O(`fmt::Write`)切换到二进制I/O，这是指把地址作为字节而不是字符串往主机发送。

二进制序列化可以很难但是我们将通过序列化每个地址为单个字节让事情变得很简单。有了这个办法我们不必担心大小端或者分帧。这个格式的缺点是单个字节只能表示最多256个不同的地址。

让我们来做一下这些改变：

``` rust
{{#include ../ci/logging/app2/src/main.rs}}
```

在你运行这个之前，你必须要在传递给链接器的参数之后添加`-Tlog.x` 。可以在Cargo的配置文件中完成它。

``` console
$ cat .cargo/config
```

``` toml
{{#include ../ci/logging/app2/.cargo/config}}
```

现在你可以运行它了!因为输出现在有一个二进制格式，我将通过管道将它输入`xxd`命令中重新格式化它为十六进制的字符串。

``` console
$ cargo run | xxd -p
```

``` text
{{#include ../ci/logging/app2/dev.out}}
```

地址是`0x00`和`0x01` 。让我们现在看下符号表。

``` console
$ cargo objdump --bin app -- -t | grep '\.log'
```

``` text
{{#include ../ci/logging/app2/dev.objdump}}
```

这是我们的字符串。我将看到它们的地址现在开始于零；这是因为我们为输出的`.log` section设置了一个起始地址。

每个变量是一个字节的大小因为我们正在使用`u8`作为它们的类型。如果我们使用类似于`u16`的东西，那么所有的地址都是偶数，我们将无法有效的使用所有的地址空间(`0...255`) 。

## 打包

你已经注意到，记录一个字符串的步骤总是一样的，因此我们可以把它们重构为一个位于自己的crate中的宏。此外，通过将I/O部分抽象成一个trait，我们还可以提高日志库的可重用部分。

``` console
$ cargo new --lib log

$ cat log/src/lib.rs
```

``` rust
{{#include ../ci/logging/log/src/lib.rs}}
```

因为这个库依赖`.log` section，所以它应该负责提供`log.x`链接器脚本，让我们来实现这一点。

``` console
$ mv log.x ../log/
```

``` console
$ cat ../log/build.rs
```

``` rust
{{#include ../ci/logging/log/build.rs}}
```

现在我们可以重构我们的应用以使用`log!`宏:

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/logging/app3/src/main.rs}}
```

不要忘记将`Cargo.toml`文件更新成依赖新的`log` crate 。

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

## 额外奖励：多个日志等级

许多日志框架提供方法让你可以使用不同的*日志等级*去记录信息。这些日志等级表示信息的严重性：“这是一个错误”，“这只是一个警告”，等等。当搜寻，比如，错误信息时，这些日志等级可以被用来过滤掉不重要的信息。

我们可以扩展我们的日志库，让它在不增加空间的情况下支持日志等级。这是我们将要做的：

我们有一个与这些信息相关的平铺的地址空间：从`0`到`255`(包含255)。为了让事情简单点，我们只想区分错误信息和警告信息。我们可以把所有的错误信息放在地址空间的开始处，并将所有的警告信息放在错误信息之后。如果解码器知道第一个警告信息的地址，那么它可以分类出消息来。这个方法可以被扩展到支持两个以上的日志等级。

让我们通过使用两个新的宏：`error!`和`warn!`替代`log`宏来测试下这个方法。

``` console
$ cat ../log/src/lib.rs
```

``` rust
{{#include ../ci/logging/log2/src/lib.rs}}
```

通过将信息放在不同的link sections中，我们可以区分错误和警告。

下一个我们必须要做的事是修改链接器脚本来将错误信息放在警告信息之前。

``` console
$ cat ../log/log.x
```

``` text
{{#include ../ci/logging/log2/log.x}}
```

我们也将错误和警告之间的边界命名为`__log_warning_start__`。这个符号的地址将是第一个警告信息的地址。

我们现在可以更新应用以使用这些新的宏。

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/logging/app4/src/main.rs}}
```

输出将不会改变太多：

``` console
$ cargo run | xxd -p
```

``` text
{{#include ../ci/logging/app4/dev.out}}
```

我们仍然在输出中获得了两个字节，但是错误被赋予了地址0，警告被赋予了地址1，即使警告被先记录。

现在看一下符号表。

```  console
$ cargo objdump --bin app -- -t | grep '\.log'
```

``` text
{{#include ../ci/logging/app4/dev.objdump}}
```

现在在`.log` section中，有了一个额外符号，`__log_warning_start__` 。这个符号的地址是第一个警告信息的地址。地址低于这个值的符号是错误，剩余的符号是警告。

使用一个合适的解码器你可以从所有这些信息获得以下人类可读的输出：

``` text
WARNING Hello, world!
ERROR Goodbye
```

---

如果你喜欢这部分，看一下[`stlog`]日志框架，它是这个想法的完整实现。

[`stlog`]: https://crates.io/crates/stlog
