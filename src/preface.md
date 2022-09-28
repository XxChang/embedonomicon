# 嵌入式宝典

嵌入式宝典带领你经历从零创造一个 `#![no_std]` 应用的过程，经历为Coterx-M微控制器搭建架构特定的功能的迭代过程。

## 目的

通过阅读这本书你将会学到

- 搭建一个 `#![no_std]` 应用。这比搭建一个 `#![no_std]` 库更复杂，因为目标系统可能没有运行一个OS(或者你的目标就是搭建一个OS!)，而且你的程序可能是目标中运行的唯一进程(或者第一个进程)。在这种情况下，程序可能需要为目标系统进行定制。

- 精细控制一个Rust程序的存储布局的技巧。你将学到链接器(linkers)，链接器脚本和 You'll learn about linkers, linker
  scripts and about the Rust features that let you control a bit of the ABI of Rust programs.

- A trick to implement default functionality that can be statically overridden (no runtime cost).

## 目标读者

This book mainly targets to two audiences:

- People that wish to bootstrap bare metal support for an architecture that the ecosystem doesn't
  yet support (e.g. Cortex-R as of Rust 1.28), or for an architecture that Rust just gained support
  for (e.g. maybe Xtensa some time in the future).

- People that are curious about the unusual implementation of *runtime* crates like [`cortex-m-rt`],
  [`msp430-rt`] and [`riscv-rt`].

[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt
[`msp430-rt`]: https://crates.io/crates/msp430-rt
[`riscv-rt`]: https://crates.io/crates/riscv-rt

## 翻译

This book has been translated by generous volunteers. If you would like your
translation listed here, please open a PR to add it.

* [Japanese](https://tomoyuki-nakabayashi.github.io/embedonomicon/)
  ([repository](https://github.com/tomoyuki-nakabayashi/embedonomicon))

## 要求

This book is self contained. The reader doesn't need to be familiar with the
Cortex-M architecture, nor is access to a Cortex-M microcontroller needed -- all
the examples included in this book can be tested in QEMU. You will, however,
need to install the following tools to run and inspect the examples in this
book:

- All the code in this book uses the 2018 edition. If you are not familiar with
  the 2018 features and idioms check the [`edition guide`].

- Rust 1.31 or a newer toolchain PLUS ARM Cortex-M compilation support.

- [`cargo-binutils`](https://github.com/japaric/cargo-binutils). v0.1.4 or newer.

- [`cargo-edit`](https://crates.io/crates/cargo-edit).

- QEMU with support for ARM emulation. The `qemu-system-arm` program must be
  installed on your computer.

- GDB with ARM support.

[`edition guide`]: https://rust-lang-nursery.github.io/edition-guide/

### Example setup

Instructions common to all OSes

``` console
$ # Rust toolchain
$ # If you start from scratch, get rustup from https://rustup.rs/
$ rustup default stable

$ # toolchain should be newer than this one
$ rustc -V
rustc 1.31.0 (abe02cefd 2018-12-04)

$ rustup target add thumbv7m-none-eabi

$ # cargo-binutils
$ cargo install cargo-binutils

$ rustup component add llvm-tools-preview

```

#### macOS

``` console
$ # arm-none-eabi-gdb
$ # you may need to run `brew tap Caskroom/tap` first
$ brew install --cask gcc-arm-embedded

$ # QEMU
$ brew install qemu
```

#### Ubuntu 16.04

``` console
$ # arm-none-eabi-gdb
$ sudo apt install gdb-arm-none-eabi

$ # QEMU
$ sudo apt install qemu-system-arm
```

#### Ubuntu 18.04 或者 Debian

``` console
$ # gdb-multiarch -- use `gdb-multiarch` when you wish to invoke gdb
$ sudo apt install gdb-multiarch

$ # QEMU
$ sudo apt install qemu-system-arm
```

#### Windows

- [arm-none-eabi-gdb](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads).
  The GNU Arm Embedded Toolchain includes GDB.

- [QEMU](https://www.qemu.org/download/#windows)

## Installing a toolchain bundle from ARM (optional step) (tested on Ubuntu 18.04)
- With the late 2018 switch from
[GCC's linker to LLD](https://rust-embedded.github.io/blog/2018-08-2x-psa-cortex-m-breakage/) for Cortex-M 
microcontrollers, [gcc-arm-none-eabi][1] is no longer 
required.  But for those wishing to use the toolchain 
anyway, install from [here][1] and follow the steps outlined below:
``` console
$ tar xvjf gcc-arm-none-eabi-8-2018-q4-major-linux.tar.bz2
$ mv gcc-arm-none-eabi-<version_downloaded> <your_desired_path> # optional
$ export PATH=${PATH}:<path_to_arm_none_eabi_folder>/bin # add this line to .bashrc to make persistent
```
[1]: https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads
