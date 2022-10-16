# 与编译器支持有关的笔记

这本书使用了一个内置的*编译器*目标，`thumbv7m-none-eabi`，Rust团队为其分发了一个 `rust-std` 组件，`rust-std` 是跟 [`core`] 和 [`std`] 一样的预先编译好的crates的集合。

[`core`]: https://doc.rust-lang.org/core/index.html
[`std`]: https://doc.rust-lang.org/std/index.html

如果你想尝试为一个不同的目标架构复制这本书的内容，你需要考虑下Rust为(编译)目标所提供的支持在什么级别。

## LLVM 支持

自Rust 1.28以来，官方的Rust编译器，`rustc`，使用LLVM生成(机器)代码。Rust对某个架构提供的最小级别的支持是在`rustc`中让它的LLVM后端可用。通过运行下面的命令，你可以看到所有的 `rustc` 通过LLVM支持的架构:

``` console
$ # 运行这个命令，你需要安装`cargo-binutils`
$ cargo objdump -- -version
LLVM (http://llvm.org/):
  LLVM version 7.0.0svn
  Optimized build.
  Default target: x86_64-unknown-linux-gnu
  Host CPU: skylake

  Registered Targets:
    aarch64    - AArch64 (little endian)
    aarch64_be - AArch64 (big endian)
    arm        - ARM
    arm64      - ARM64 (little endian)
    armeb      - ARM (big endian)
    hexagon    - Hexagon
    mips       - Mips
    mips64     - Mips64 [experimental]
    mips64el   - Mips64el [experimental]
    mipsel     - Mipsel
    msp430     - MSP430 [experimental]
    nvptx      - NVIDIA PTX 32-bit
    nvptx64    - NVIDIA PTX 64-bit
    ppc32      - PowerPC 32
    ppc64      - PowerPC 64
    ppc64le    - PowerPC 64 LE
    sparc      - Sparc
    sparcel    - Sparc LE
    sparcv9    - Sparc V9
    systemz    - SystemZ
    thumb      - Thumb
    thumbeb    - Thumb (big endian)
    wasm32     - WebAssembly 32-bit
    wasm64     - WebAssembly 64-bit
    x86        - 32-bit X86: Pentium-Pro and above
    x86-64     - 64-bit X86: EM64T and AMD64
```

如果LLVM支持了你感兴趣的架构，但是构建的`rustc`没有使能其后端(自Rust 1.28以来AVR就是这样的情况)，那么你将需要修改Rust源码以启用它。PR [rust-lang/rust#52787] 的前两个commits能让你知道需要做的改变。

[rust-lang/rust#52787]: https://github.com/rust-lang/rust/pull/52787

换句话说，如果LLVM不支持这个架构，但是LLVM的一个分支支持，在编译`rustc`之前你将需要使用这个分支替代原来的LLVM 。Rust编译系统允许这么做且理论上，它应该只需要修改`llvm`的子模组让它指向那个分支就可以了。

如果只有一些供应商提供的GCC支持你的目标架构，你可以选择使用[`mrustc`]，一个非官方的Rust编译器，去把你的Rust程序翻译成C代码然后使用GCC编译它。

[`mrustc`]: https://github.com/thepowersgang/mrustc

## 内置的目标

一个编译目标不仅是它的架构。每个目标都有一个与它关联的[规范]，其中描述了它的架构，它的操作系统和默认的链接器。

[规范]: https://github.com/rust-lang/rfcs/blob/master/text/0131-target-specification.md

Rust编译器知道几个目标。这些被*内置进*编译器里且可以通过下列的命令被列出来:

``` console
$ rustc --print target-list | column
aarch64-fuchsia                   mipsisa32r6el-unknown-linux-gnu
aarch64-linux-android             mipsisa64r6-unknown-linux-gnuabi64
aarch64-pc-windows-msvc           mipsisa64r6el-unknown-linux-gnuabi64
aarch64-unknown-cloudabi          msp430-none-elf
aarch64-unknown-freebsd           nvptx64-nvidia-cuda
aarch64-unknown-hermit            powerpc-unknown-linux-gnu
aarch64-unknown-linux-gnu         powerpc-unknown-linux-gnuspe
aarch64-unknown-linux-musl        powerpc-unknown-linux-musl
aarch64-unknown-netbsd            powerpc-unknown-netbsd
aarch64-unknown-none              powerpc-wrs-vxworks
aarch64-unknown-none-softfloat    powerpc-wrs-vxworks-spe
aarch64-unknown-openbsd           powerpc64-unknown-freebsd
aarch64-unknown-redox             powerpc64-unknown-linux-gnu
aarch64-uwp-windows-msvc          powerpc64-unknown-linux-musl
aarch64-wrs-vxworks               powerpc64-wrs-vxworks
arm-linux-androideabi             powerpc64le-unknown-linux-gnu
arm-unknown-linux-gnueabi         powerpc64le-unknown-linux-musl
arm-unknown-linux-gnueabihf       riscv32i-unknown-none-elf
arm-unknown-linux-musleabi        riscv32imac-unknown-none-elf
arm-unknown-linux-musleabihf      riscv32imc-unknown-none-elf
armebv7r-none-eabi                riscv64gc-unknown-linux-gnu
armebv7r-none-eabihf              riscv64gc-unknown-none-elf
armv4t-unknown-linux-gnueabi      riscv64imac-unknown-none-elf
armv5te-unknown-linux-gnueabi     s390x-unknown-linux-gnu
armv5te-unknown-linux-musleabi    sparc-unknown-linux-gnu
armv6-unknown-freebsd             sparc64-unknown-linux-gnu
armv6-unknown-netbsd-eabihf       sparc64-unknown-netbsd
armv7-linux-androideabi           sparc64-unknown-openbsd
armv7-unknown-cloudabi-eabihf     sparcv9-sun-solaris
armv7-unknown-freebsd             thumbv6m-none-eabi
armv7-unknown-linux-gnueabi       thumbv7a-pc-windows-msvc
armv7-unknown-linux-gnueabihf     thumbv7em-none-eabi
armv7-unknown-linux-musleabi      thumbv7em-none-eabihf
armv7-unknown-linux-musleabihf    thumbv7m-none-eabi
armv7-unknown-netbsd-eabihf       thumbv7neon-linux-androideabi
armv7-wrs-vxworks-eabihf          thumbv7neon-unknown-linux-gnueabihf
armv7a-none-eabi                  thumbv7neon-unknown-linux-musleabihf
armv7a-none-eabihf                thumbv8m.base-none-eabi
armv7r-none-eabi                  thumbv8m.main-none-eabi
armv7r-none-eabihf                thumbv8m.main-none-eabihf
asmjs-unknown-emscripten          wasm32-unknown-emscripten
hexagon-unknown-linux-musl        wasm32-unknown-unknown
i586-pc-windows-msvc              wasm32-wasi
i586-unknown-linux-gnu            x86_64-apple-darwin
i586-unknown-linux-musl           x86_64-fortanix-unknown-sgx
i686-apple-darwin                 x86_64-fuchsia
i686-linux-android                x86_64-linux-android
i686-pc-windows-gnu               x86_64-linux-kernel
i686-pc-windows-msvc              x86_64-pc-solaris
i686-unknown-cloudabi             x86_64-pc-windows-gnu
i686-unknown-freebsd              x86_64-pc-windows-msvc
i686-unknown-haiku                x86_64-rumprun-netbsd
i686-unknown-linux-gnu            x86_64-sun-solaris
i686-unknown-linux-musl           x86_64-unknown-cloudabi
i686-unknown-netbsd               x86_64-unknown-dragonfly
i686-unknown-openbsd              x86_64-unknown-freebsd
i686-unknown-uefi                 x86_64-unknown-haiku
i686-uwp-windows-gnu              x86_64-unknown-hermit
i686-uwp-windows-msvc             x86_64-unknown-hermit-kernel
i686-wrs-vxworks                  x86_64-unknown-illumos
mips-unknown-linux-gnu            x86_64-unknown-l4re-uclibc
mips-unknown-linux-musl           x86_64-unknown-linux-gnu
mips-unknown-linux-uclibc         x86_64-unknown-linux-gnux32
mips64-unknown-linux-gnuabi64     x86_64-unknown-linux-musl
mips64-unknown-linux-muslabi64    x86_64-unknown-netbsd
mips64el-unknown-linux-gnuabi64   x86_64-unknown-openbsd
mips64el-unknown-linux-muslabi64  x86_64-unknown-redox
mipsel-unknown-linux-gnu          x86_64-unknown-uefi
mipsel-unknown-linux-musl         x86_64-uwp-windows-gnu
mipsel-unknown-linux-uclibc       x86_64-uwp-windows-msvc
mipsisa32r6-unknown-linux-gnu     x86_64-wrs-vxworks
```

使用下列的命令你可以打印出某个目标的规范:

``` console
$ rustc +nightly -Z unstable-options --print target-spec-json --target thumbv7m-none-eabi
{
  "abi-blacklist": [
    "stdcall",
    "fastcall",
    "vectorcall",
    "thiscall",
    "win64",
    "sysv64"
  ],
  "arch": "arm",
  "data-layout": "e-m:e-p:32:32-i64:64-v128:64:128-a:0:32-n32-S64",
  "emit-debug-gdb-scripts": false,
  "env": "",
  "executables": true,
  "is-builtin": true,
  "linker": "arm-none-eabi-gcc",
  "linker-flavor": "gcc",
  "llvm-target": "thumbv7m-none-eabi",
  "max-atomic-width": 32,
  "os": "none",
  "panic-strategy": "abort",
  "relocation-model": "static",
  "target-c-int-width": "32",
  "target-endian": "little",
  "target-pointer-width": "32",
  "vendor": ""
}
```

如果这些内置的目标都不适合你的目标系统，你将必须通过使用JSON格式编写你自己的目标规范来生成一个自制的目标，在[下个章节][custom-target]中会讲到。

[custom-target]: ./custom-target.md

## `rust-std` 组件

Rust团队通过`rustup`为某些内置的目标分发`rust-std`组件。这个组件是跟 [`core`] 和 [`std`] 一样的预先编译好的crates的集合，它要求交叉编译。

通过运行下列的命令，你可以找到具有一个`rust-std`组件的目标列表:

``` console
$ rustup target list | column
aarch64-apple-ios                       mipsel-unknown-linux-musl
aarch64-fuchsia                         nvptx64-nvidia-cuda
aarch64-linux-android                   powerpc-unknown-linux-gnu
aarch64-pc-windows-msvc                 powerpc64-unknown-linux-gnu
aarch64-unknown-linux-gnu               powerpc64le-unknown-linux-gnu
aarch64-unknown-linux-musl              riscv32i-unknown-none-elf
aarch64-unknown-none                    riscv32imac-unknown-none-elf
aarch64-unknown-none-softfloat          riscv32imc-unknown-none-elf
arm-linux-androideabi                   riscv64gc-unknown-linux-gnu
arm-unknown-linux-gnueabi               riscv64gc-unknown-none-elf
arm-unknown-linux-gnueabihf             riscv64imac-unknown-none-elf
arm-unknown-linux-musleabi              s390x-unknown-linux-gnu
arm-unknown-linux-musleabihf            sparc64-unknown-linux-gnu
armebv7r-none-eabi                      sparcv9-sun-solaris
armebv7r-none-eabihf                    thumbv6m-none-eabi
armv5te-unknown-linux-gnueabi           thumbv7em-none-eabi
armv5te-unknown-linux-musleabi          thumbv7em-none-eabihf
armv7-linux-androideabi                 thumbv7m-none-eabi
armv7-unknown-linux-gnueabi             thumbv7neon-linux-androideabi
armv7-unknown-linux-gnueabihf           thumbv7neon-unknown-linux-gnueabihf
armv7-unknown-linux-musleabi            thumbv8m.base-none-eabi
armv7-unknown-linux-musleabihf          thumbv8m.main-none-eabi
armv7a-none-eabi                        thumbv8m.main-none-eabihf
armv7r-none-eabi                        wasm32-unknown-emscripten
armv7r-none-eabihf                      wasm32-unknown-unknown
asmjs-unknown-emscripten                wasm32-wasi
i586-pc-windows-msvc                    x86_64-apple-darwin
i586-unknown-linux-gnu                  x86_64-apple-ios
i586-unknown-linux-musl                 x86_64-fortanix-unknown-sgx
i686-linux-android                      x86_64-fuchsia
i686-pc-windows-gnu                     x86_64-linux-android
i686-pc-windows-msvc                    x86_64-pc-windows-gnu
i686-unknown-freebsd                    x86_64-pc-windows-msvc
i686-unknown-linux-gnu                  x86_64-rumprun-netbsd
i686-unknown-linux-musl                 x86_64-sun-solaris
mips-unknown-linux-gnu                  x86_64-unknown-cloudabi
mips-unknown-linux-musl                 x86_64-unknown-freebsd
mips64-unknown-linux-gnuabi64           x86_64-unknown-linux-gnu (default)
mips64-unknown-linux-muslabi64          x86_64-unknown-linux-gnux32
mips64el-unknown-linux-gnuabi64         x86_64-unknown-linux-musl
mips64el-unknown-linux-muslabi64        x86_64-unknown-netbsd
mipsel-unknown-linux-gnu                x86_64-unknown-redox
```

如果你的目标没有`rust-std`组件，或者你正在使用一个自制的目标，那么你必须要使用一个开发版的工具链去构建标准库。看下一页，关于[构建自制目标][use-target-file]。

[use-target-file]: ./custom-target.md#use-the-target-file
[xargo]: https://github.com/japaric/xargo
