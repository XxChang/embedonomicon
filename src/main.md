# 一个 `main` 接口

我们现在有了一个可以工作的最小的程序，但是我们需要用一个方法将它打包起来，这个方法可以让终端用户在其上搭建安全的程序。这部分，我们将实现一个 `main` 接口，就像一个标准的Rust程序所用的那样。

首先，我们将把我们的binary crate转换成一个library crate:

``` console
$ mv src/main.rs src/lib.rs
```

然后把它重命名为 `rt`，其表示"runtime" 。

``` console
$ sed -i s/app/rt/ Cargo.toml

$ head -n4 Cargo.toml
```

``` toml
{{#include ../ci/main/rt/Cargo.toml:1:4}}
```

第一个改变是让重置处理函数调用一个外部的 `main` 函数:

``` console
$ head -n13 src/lib.rs
```

``` rust
{{#include ../ci/main/rt/src/lib.rs:1:13}}
```

我们也去掉了 `#![no_main]` 属性，因为它对library crates没有影响。

> 这个阶段存在一个正交问题: `rt`库应该提供一个标准的恐慌时行为吗，
> 或者它不应该提供一个 `#[panic_handler]` 函数而让终端用户去选
> 择恐慌时行为吗？这个文档将不会深入这个问题，且为了简洁性，在`rt` crate
> 中留了一个空的 `#[panic_handler]` 函数。然而，我们想告诉读者存在其它选择。

第二个改变涉及到给应用crate提供我们之前写的链接器脚本。链接器将会在库搜索路径(`-L`)和它被调用的文件夹中寻找链接器脚本。应用crate不应该需要将`link.x`的副本挪来挪去所以我们将使用一个[build script]让 `rt` crate 将链接器脚本放到库搜索路径中。

[build script]: https://doc.rust-lang.org/cargo/reference/build-scripts.html

``` console
$ # 在`rt`的根目录中生成一个带有这些内容的 build.rs 文件
$ cat build.rs
```

``` rust
{{#include ../ci/main/rt/build.rs}}
```

现在用户可以写一个暴露了`main`符号的应用了，且将它链接到`rt` crate上。`rt` 将负责给予程序正确的存储布局。

``` console
$ cd ..

$ cargo new --edition 2018 --bin app

$ cd app

$ # 修改Cargo.toml将`rt` crate包含进来作为一个依赖
$ tail -n2 Cargo.toml
```

``` toml
{{#include ../ci/main/app/Cargo.toml:7:8}}
```

``` console
$ # 拷贝整个设置了一个默认目标的config文件并修改链接器命令
$ cp -r ../rt/.cargo .

$ # 把 `main.rs` 的内容改成
$ cat src/main.rs
```

``` rust
{{#include ../ci/main/app/src/main.rs}}
```

反汇编将是相似的，除了现在包含了用户的`main`函数。

``` console
$ cargo objdump --bin app -- -d --no-show-raw-insn
```

``` text
{{#include ../ci/main/app/app.objdump}}
```

## 把它变成类型安全的

`main` 接口工作了，但是它容易出错。比如，用户可以把`main`写成一个non-divergent function，它们将不会出现编译时错误并带来未定义的行为(编译器将会错误优化这个程序)。

我们通过暴露一个宏给用户而不是符号接口可以添加类型安全性。在 `rt` crate 中，我们可以写这个宏:

``` console
$ tail -n12 ../rt/src/lib.rs
```

``` rust
{{#include ../ci/main/rt/src/lib.rs:25:37}}
```

然后应用的作者可以像这样调用它:

``` console
$ cat src/main.rs
```

``` rust
{{#include ../ci/main/app2/src/main.rs}}
```

如果作者把`main`的签名改成non divergent function，比如 `fn` ，将会出现一个错误。

## main之前的生活

`rt` 看起来不错了，但是它的功能不够完整！用它编写的应用不能使用 `static` 变量或者字符串字面值，因为 `rt` 的链接器脚本没有定义标准的`.bss`，`.data` 和 `.rodata` sections 。让我们修复它！

第一步是在链接器脚本中定义这些sections:

``` console
$ # 只展示文件的一小块
$ sed -n 25,46p ../rt/link.x
```

``` text
{{#include ../ci/main/rt/link.x:25:46}}
```

它们只是重新导出input sections并指定每个output section将会进入哪个内存区域。

有了这些改变，下面的程序将会编译:

``` rust
{{#include ../ci/main/app3/src/main.rs}}
```

然而如果你在正真的硬件上运行这个程序并调试它，你将发现到达`main`时，`static` 变量 `BSS` 和 `DATA` 没有 `0` 和 `1` 值。反而，这些变量将是垃圾值。问题是在设备上电之后，RAM的内容是随机的。如果你在QEMU中运行这个程序，你将看不到这个影响。

在目前的情况下，如果你的程序在对`static`变量执行一个写入之前，读取任何 `static` 变量，那么你的程序会出现未定义的行为。让我们通过在调用`main`之前初始化所有的`static`变量来修复它。

我们需要修改下链接器脚本去进行RAM初始化:

``` console
$ # 只展示文件的一块
$ sed -n 25,52p ../rt/link.x
```

``` text
{{#include ../ci/main/rt2/link.x:25:52}}
```

让我们深入下细节:

``` text
{{#include ../ci/main/rt2/link.x:38}}
```

``` text
{{#include ../ci/main/rt2/link.x:40}}
```

``` text
{{#include ../ci/main/rt2/link.x:45}}
```

``` text
{{#include ../ci/main/rt2/link.x:47}}
```

我们将符号关联到`.bss`和`.data` sections的开始和末尾地址，我之后将会从Rust代码中使用。

``` text
{{#include ../ci/main/rt2/link.x:43}}
```

我们将`.data` section的加载内存地址(LMA)设置成`.rodata` section的末尾处。`.data` 包含具有一个非零的初始值的`static`变量；`.data` section的虚拟内存地址(VMA)在RAM中的某处 -- 这是`static`变量所在的地方。这些`static`变量的初始值，然而，必须被分配在非易失存储中(Flash)；LMA是Flash中存放这些初始值的地方。

``` text
{{#include ../ci/main/rt2/link.x:50}}
```

最后，我们将一个符号和`.data`的LMA关联起来。我们可以从Rust代码引用我们在链接器脚本中生成的符号。这些符号的*地址*[^1]是 `.bss` 和 `.data` sections的边界处。

下面展示的是更新了的重置处理函数:

``` console
$ head -n32 ../rt/src/lib.rs
```

``` rust
{{#include ../ci/main/rt2/src/lib.rs:1:31}}
```

现在终端用户可以直接地和间接地使用`static`变量而不会导致未定义的行为!

> 我们上面展示的代码中，内存的初始化按照一种逐字节的方式进行。可以强迫 `.bss` 和 `.data` section对齐，比如，四个字节。然后Rust代码可以利用这个事实去执行逐字的初始化而不需要对齐检查。如果你想知道这是如何做到，看下 [`cortex-m-rt`] crate 。

[`cortex-m-rt`]: https://github.com/japaric/cortex-m-rt/tree/v0.5.1

[^1]: 必须在这使用链接器脚本符号这件事会让人疑惑且反直觉。可以在[这里](https://stackoverflow.com/a/40392131)找到与这个怪现象有关的详尽解释。
