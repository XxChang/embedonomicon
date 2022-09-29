# 稳定版中的汇编

> 注意: 自Rust 1.59以来，*inline* 汇编(`asm!`)和 *free form* 汇编(`global_asm!`)
> 变得稳定了。但是因为现存的crates需要花点时间来消化这种变化，并且了解我们在历史中曾用过的处
> 理汇编的方法对我来说也有好处，所以我们保留了这一章。

到目前位置我们已经成功地引导了设备和处理中断而没使用一行汇编。这真是一个壮举!但是取决于你的目标架构你可能需要一些汇编才能达成目的。也有一些操作像是上下文切换需要汇编，等等。

问题是 *inline* 汇编(`asm!`) 和 *free form* 汇编(`global_asm!`)是不稳定的，没法估计它们什么时候将变得稳定，所以你不能在稳定版中使用它们。这不是一个演示，因为这里记录的是一些变通的方法。

为了激发对这章的兴趣，我们将修改下`HardFault`处理函数以提供关于产生了异常的栈帧的信息。

这是我们想要做的:

我们将让`rt` crate
Instead of letting the user directly put their `HardFault` handler in the vector
table we'll make the `rt` crate put a trampoline to the user-defined `HardFault`
handler in the vector table.

``` console
$ tail -n36 ../rt/src/lib.rs
```

``` rust
{{#include ../ci/asm/rt/src/lib.rs:61:96}}
```

This trampoline will read the stack pointer and then call the user `HardFault`
handler. The trampoline will have to be written in assembly:

``` armasm
{{#include ../ci/asm/rt/asm.s:5:6}}
```

Due to how the ARM ABI works this sets the Main Stack Pointer (MSP) as the first
argument of the `HardFault` function / routine. This MSP value also happens to
be a pointer to the registers pushed to the stack by the exception. With these
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

We can confirm that the vector table contains a pointer to `HardFaultTrampoline`
by writing a very simple program.

``` rust
{{#include ../ci/asm/app/src/main.rs}}
```

Here's the disassembly. Look at the address of `HardFaultTrampoline`.

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

We'll do something similar to produce an archive.

``` console
$ # most of flags `cc` uses have no effect when assembling so we drop them
$ arm-none-eabi-as -march=armv7-m asm.s -o asm.o

$ ar crs librt.a asm.o

$ arm-none-eabi-objdump -Cd librt.a
```

``` text
{{#include ../ci/asm/rt2/librt.objdump}}
```

Next we modify the build script to bundle this archive with the `rt` rlib.

``` console
$ cat ../rt/build.rs
```

``` rust
{{#include ../ci/asm/rt2/build.rs}}
```

Now we can test this new version against the simple program from before and
we'll get the same output.

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
