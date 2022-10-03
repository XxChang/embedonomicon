# 稳定版中的汇编

> 注意: 自Rust 1.59以来，*inline* 汇编(`asm!`)和 *free form* 汇编(`global_asm!`)
> 变得稳定了。但是因为现存的crates需要花点时间来消化这种变化，并且了解我们在历史中曾用过的处
> 理汇编的方法对我来说也有好处，所以我们保留了这一章。

到目前位置我们已经成功地引导了设备和处理中断而没使用一行汇编。这真是一个壮举!但是取决于你的目标架构你可能需要一些汇编才能达成目的。也有一些操作像是上下文切换需要汇编，等等。

问题是 *inline* 汇编(`asm!`) 和 *free form* 汇编(`global_asm!`)是不稳定的，没法估计它们什么时候将变得稳定，所以你不能在稳定版中使用它们。这不是一个演示，因为这里记录的是一些变通的方法。

为了激发对这章的兴趣，我们将修改下`HardFault`处理函数以提供关于产生了异常的栈帧的信息。

这是我们想要做的:

我们将让`rt` crate在向量表中放置一个跳向用户定义的`HardFault`的跳板，而不是让用户直接将它们的`HardFault`处理函数放在向量表中。

``` console
$ tail -n36 ../rt/src/lib.rs
```

``` rust
{{#include ../ci/asm/rt/src/lib.rs:61:96}}
```

这个跳板将读取栈指针并调用用户的`HardFault`处理函数。这个跳板必须要用汇编来写:

``` armasm
{{#include ../ci/asm/rt/asm.s:5:6}}
```

由于ARM ABI的工作原理，它将主堆栈指针(MSP)设置为 `HardFault` 函数/例程的第一个参数。这个MSP值碰巧也是一个指针，其指向被异常推进栈中的寄存器。
With these
changes the user `HardFault` handler must now have signature
`fn(&StackedRegisters) -> !`.

## `.s` 文件

One approach to stable assembly is to write the assembly in an external file:

``` console
$ cat ../rt/asm.s
```

``` armasm
{{#include ../ci/asm/rt/asm.s}}
```

And use the `cc` crate in the build script of the `rt` crate to assemble that
file into an object file (`.o`) and then into an archive (`.a`).

``` console
$ cat ../rt/build.rs
```

``` rust
{{#include ../ci/asm/rt/build.rs}}
```

``` console
$ tail -n2 ../rt/Cargo.toml
```

``` toml
{{#include ../ci/asm/rt/Cargo.toml:7:8}}
```

And that's it!

通过编写一个非常简单的程序我们可以确认向量表包含一个指向 `HardFaultTrampoline` 的指针。

``` rust
{{#include ../ci/asm/app/src/main.rs}}
```

这是反汇编。我们看下 `HardFaultTrampoline` 的地址。

``` console
$ cargo objdump --bin app --release -- -d --no-show-raw-insn --print-imm-hex
```

``` text
{{#include ../ci/asm/app/release.objdump}}
```

> **注意:** 为了让这个反汇编更小我注释掉了RAM的初始化

现在看下向量表。第四项应该是`HardFaultTrampoline`的地址加一。

``` console
$ cargo objdump --bin app --release -- -s -j .vector_table
```

``` text
{{#include ../ci/asm/app/release.vector_table}}
```

## `.o` / `.a` 文件

The downside of using the `cc` crate is that it requires some assembler program
on the build machine. For example when targeting ARM Cortex-M the `cc` crate
uses `arm-none-eabi-gcc` as the assembler.

Instead of assembling the file on the build machine we can ship a pre-assembled
file with the `rt` crate. That way no assembler program is required on the build
machine. However, you would still need an assembler on the machine that packages
and publishes the crate.

There's not much difference between an assembly (`.s`) file and its *compiled*
version: the object (`.o`) file. The assembler doesn't do any optimization; it
simply chooses the right object file format for the target architecture.

Cargo provides support for bundling archives (`.a`) with crates. We can package
object files into an archive using the `ar` command and then bundle the archive
with the crate. In fact, this what the `cc` crate does; you can see the commands
it invoked by searching for a file named `output` in the `target` directory.

``` console
$ grep running $(find target -name output)
```

``` text
running: "arm-none-eabi-gcc" "-O0" "-ffunction-sections" "-fdata-sections" "-fPIC" "-g" "-fno-omit-frame-pointer" "-mthumb" "-march=armv7-m" "-Wall" "-Wextra" "-o" "/tmp/app/target/thumbv7m-none-eabi/debug/build/rt-6ee84e54724f2044/out/asm.o" "-c" "asm.s"
running: "ar" "crs" "/tmp/app/target/thumbv7m-none-eabi/debug/build/rt-6ee84e54724f2044/out/libasm.a" "/home/japaric/rust-embedded/embedonomicon/ci/asm/app/target/thumbv7m-none-eabi/debug/build/rt-6ee84e54724f2044/out/asm.o"
```

``` console
$ grep cargo $(find target -name output)
```

``` tetx
cargo:rustc-link-search=/tmp/app/target/thumbv7m-none-eabi/debug/build/rt-6ee84e54724f2044/out
cargo:rustc-link-lib=static=asm
cargo:rustc-link-search=native=/tmp/app/target/thumbv7m-none-eabi/debug/build/rt-6ee84e54724f2044/out
```

我们将做一些类似的事来生成一个归档文件。

``` console
$ # `cc` 使用的大多数标志在汇编时没有影响因此我们丢弃它们
$ arm-none-eabi-as -march=armv7-m asm.s -o asm.o

$ ar crs librt.a asm.o

$ arm-none-eabi-objdump -Cd librt.a
```

``` text
{{#include ../ci/asm/rt2/librt.objdump}}
```

接下来我们修改build script以把这个归档文件和`rt` rlib绑一起。

``` console
$ cat ../rt/build.rs
```

``` rust
{{#include ../ci/asm/rt2/build.rs}}
```

现在我们可以用以前的简单程序测试这个新版本，我们将得到相同的输出

``` console
$ cargo objdump --bin app --release -- -d --no-show-raw-insn --print-imm-hex
```

``` text
{{#include ../ci/asm/app2/release.objdump}}
```

> **注意**: 像之前一样我已经注释掉了RAM的初始化以让反汇编变得更小。

``` console
$ cargo objdump --bin app --release -- -s -j .vector_table
```

``` text
{{#include ../ci/asm/app2/release.vector_table}}
```

The downside of shipping pre-assembled archives is that, in the worst case
scenario, you'll need to ship one build artifact for each compilation target
your library supports.
