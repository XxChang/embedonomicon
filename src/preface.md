# 嵌入式宝典

嵌入式宝典将会带领你从零创造一个 `#![no_std]` 应用，经历为Coterx-M微控制器搭建功能的迭代过程。

## 目的

通过阅读这本书你将会学到

- 搭建一个 `#![no_std]` 应用。这比搭建一个 `#![no_std]` 库更复杂，因为目标系统可能没有运行操作系统(或者你的目标就是搭建一个操作系统!)，而且你的程序可能是目标中运行的唯一一个进程(或者第一个进程)。在这种情况下，程序可能需要为目标系统进行定制。

- 精细控制一个Rust程序的存储布局的技巧。你将学到链接器(linkers)，链接器脚本和可以用来控制Rust程序中的某些ABI的 Rust features 。

- 如何实现可以被静态重载(没有运行时消耗)的默认函数。

## 目标读者

这本书主要面向两个读者:

- 希望为一个生态系统还没有支持的架构提供板级支持(比如，自Rust 1.28以来的Cortex-R)，或者为一个刚获得Rust支持的架构提供支持(比如 未来可能有Xtensa)

- 对像是 [`cortex-m-rt`]，[`msp430-rt`] 和 [`riscv-rt`] 这样的 *runtime* 库的不寻常的实现感到好奇的人。 

[`cortex-m-rt`]: https://crates.io/crates/cortex-m-rt
[`msp430-rt`]: https://crates.io/crates/msp430-rt
[`riscv-rt`]: https://crates.io/crates/riscv-rt

## 翻译

这本书已经被慷慨的志愿者们翻译了。如果你想要你的翻译被列在这里，请打开一个PR添加它。

* [日文](https://tomoyuki-nakabayashi.github.io/embedonomicon/)
  ([repository](https://github.com/tomoyuki-nakabayashi/embedonomicon))

* [中文](https://xxchang.github.io/embedonomicon/)
  ([repository](https://github.com/xxchang/embedonomicon))

## 要求

这本书是自洽的。读者不需要熟悉Cortex-M架构，也不需要一个Cortex-M微控制器 -- 这本书里包含
的所有例子都能在QEMU中测试。然而，你需要安装下面的工具来运行和检查这本书中的示例:

- 这本书中所有代码使用的是2018版的Rust。如果你不熟悉2018的特性和术语，阅读 [`edition guide`] 。

- Rust 1.31 或者更新的具有ARM Cortex-M编译支持的工具链。

- [`cargo-binutils`](https://github.com/japaric/cargo-binutils). v0.1.4 或者更新的版本。

- [`cargo-edit`](https://crates.io/crates/cargo-edit).

- 有ARM仿真支持的QEMU。`qemu-system-arm` 程序必须被安装在你的电脑上。

- 有ARM支持的GDB 。

[`edition guide`]: https://rust-lang-nursery.github.io/edition-guide/

### 安装示例

所有操作系统通用的指令

``` console
$ # Rust 工具链
$ # 如果你是从零开始，从 https://rustup.rs/ 获取rustup
$ rustup default stable

$ # 工具链应该比这个更新
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
$ # 你可能需要先运行 `brew tap Caskroom/tap`
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
$ # gdb-multiarch -- 当你希望启动gdb时，使用 `gdb-multiarch`
$ sudo apt install gdb-multiarch

$ # QEMU
$ sudo apt install qemu-system-arm
```

#### Windows

- [arm-none-eabi-gdb](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads).
  The GNU Arm Embedded Toolchain 包括 GDB 。

- [QEMU](https://www.qemu.org/download/#windows)

## 从ARM安装一个工具链(可选的步骤) (在Ubuntu 18.04上测试过)
- 2018年之后，对于Cortex-M微控制器[GCC的链接器换成了LLD](https://rust-embedded.github.io/blog/2018-08-2x-psa-cortex-m-breakage/)，[gcc-arm-none-eabi][1] 不再需要了。但是对于那些想使用这个工具链的人，可以从[这里][1]安装并按照下面的步骤设置:
``` console
$ tar xvjf gcc-arm-none-eabi-8-2018-q4-major-linux.tar.bz2
$ mv gcc-arm-none-eabi-<version_downloaded> <your_desired_path> # 可选
$ export PATH=${PATH}:<path_to_arm_none_eabi_folder>/bin # 把这行添加到 .bashrc 使它永久有效
```
[1]: https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads
