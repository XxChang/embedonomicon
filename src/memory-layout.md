# 存储布局

下一步是确保程序有正确的存储布局，因此目标系统才可以执行它。在我们的例子里，我将使用一个虚拟的Cortex-M3微控制器: [LM3S6965] 。我们的程序是运行在设备上的唯一一个进程，因此它也必须要负责初始化设备。

## 背景信息

[LM3S6965]: http://www.ti.com/product/LM3S6965

Cortex-M 设备需要[向量表]放在它们[代码区]的开始处。
向量表是一组指针；启动设备需要前两个指针，剩下来的指针都与异常有关。我们现在忽略它们。

[代码区]: https://developer.arm.com/docs/dui0552/latest/the-cortex-m3-processor/memory-model
[向量表]: https://developer.arm.com/docs/dui0552/latest/the-cortex-m3-processor/exception-model/vector-table

链接器决定程序的最终存储布局，但是我们可以用[链接器脚本]对它进行一些控制。链接器提供的控制布局的精细度在*sections*层级。一个section是分布在连续存储中的*符号*的集合。符号，依次，可以是数据(一个静态变量)，或者指令(一个Rust函数)。

[链接器脚本]: https://sourceware.org/binutils/docs/ld/Scripts.html

每一个符号有一个由编译器分配的名字。自Rust 1.28以来，Rust编译器分配给符号的名字都是这样的格式:
`_ZN5krate6module8function17he1dfc17c86fe16daE`，其展开是 `krate::module::function::he1dfc17c86fe16da` 其中 `krate::module::function` 是函数或者变量的路径，`he1dfc17c86fe16da` 某个哈希值。Rust编译器将把每个符号放进它自己的独有的section中；比如之前提到的符号将被放进一个名为 `.text._ZN5krate6module8function17he1dfc17c86fe16daE` 的section中。

这些编译器产生的符号和section名在不同的Rust编译器发布版中不保证是不变的。然而，语言让我们可以使用这些属性控制符号名和放置的section:

- `#[export_name = "foo"]` 把符号名设置成 `foo` 。
- `#[no_mangle]` 意思是: 使用函数或者变量名(不是它的全路径)作为它的符号名。
  `#[no_mangle] fn bar()` 将产生一个名为 `bar` 的符号。
- `#[link_section = ".bar"]` 把符号放进一个名为 `.bar` 的section中。

有了这些属性，我们可以暴露一个程序的稳定的ABI，并在链接器脚本中使用它。

## Rust 部分

像上面提到的，对于Cortex-M设备，我们需要修改向量表的前两项。第一个，栈指针的初始值，只能使用链接器脚本修改。第二个，重置向量，需要在Rust代码中生成并使用链接器脚本放置到正确的地方。

重置向量是一个指向重置处理函数的指针。重置处理函数是在一个系统重启后设备将会执行的函数，或者第一次上电后。重置处理函数总是硬件调用栈中的第一个栈帧；从它返回是未定义的行为，因为没有其它栈帧可以给它返回。通过让它变成一个divergent function，我们可以强调这个重置处理函数从来不会返回，divergent function的签名是 `fn(/* .. */) -> !` 。

``` rust
{{#include ../ci/memory-layout/src/main.rs:7:19}}
```

The hardware expects a certain format here, to which we adhere by using `extern "C"` to tell the
compiler to lower the function using the C ABI, instead of the Rust ABI, which is unstable.

To refer to the reset handler and reset vector from the linker script, we need them to have a stable
symbol name so we use `#[no_mangle]`. We need fine control over the location of `RESET_VECTOR`, so we
place it in a known section, `.vector_table.reset_vector`. The exact location of the reset handler
itself, `Reset`, is not important. We just stick to the default compiler generated section.

The linker will ignore symbols with internal linkage (also known as internal symbols) while traversing
the list of input object files, so we need our two symbols to have external linkage. The only way to
make a symbol external in Rust is to make its corresponding item public (`pub`) and *reachable* (no
private module between the item and the root of the crate).

## The linker script side

A minimal linker script that places the vector table in the correct location is shown below. Let's
walk through it.

``` console
$ cat link.x
```

``` text
{{#include ../ci/memory-layout/link.x}}
```

### `MEMORY`

This section of the linker script describes the location and size of blocks of memory in the target.
Two memory blocks are defined: `FLASH` and `RAM`; they correspond to the physical memory available
in the target. The values used here correspond to the LM3S6965 microcontroller.

### `ENTRY`

Here we indicate to the linker that the reset handler, whose symbol name is `Reset`, is the
*entry point* of the program. Linkers aggressively discard unused sections. Linkers consider the
entry point and functions called from it as *used* so they won't discard them. Without this line,
the linker would discard the `Reset` function and all subsequent functions called from it.

### `EXTERN`

Linkers are lazy; they will stop looking into the input object files once they have found all the
symbols that are recursively referenced from the entry point. `EXTERN` forces the linker to look
for `EXTERN`'s argument even after all other referenced symbols have been found. As a rule of thumb,
if you need a symbol that's not called from the entry point to always be present in the output binary,
you should use `EXTERN` in conjunction with `KEEP`.

### `SECTIONS`

This part describes how sections in the input object files (also known as *input sections*) are to be arranged
in the sections of the output object file (also known as output sections) or if they should be discarded. Here
we define two output sections:

``` text
  .vector_table ORIGIN(FLASH) : { /* .. */ } > FLASH
```

`.vector_table` contains the vector table and is located at the start of `FLASH` memory.

``` text
  .text : { /* .. */ } > FLASH
```

`.text` contains the program subroutines and is located somewhere in `FLASH`. Its start
address is not specified, but the linker will place it after the previous output section,
`.vector_table`.

The output `.vector_table` section contains:

``` text
{{#include ../ci/memory-layout/link.x:18:19}}
```

We'll place the (call) stack at the end of RAM (the stack is *full descending*; it grows towards
smaller addresses) so the end address of RAM will be used as the initial Stack Pointer (SP) value.
That address is computed in the linker script itself using the information we entered for the `RAM`
memory block.

```
{{#include ../ci/memory-layout/link.x:21:22}}
```

Next, we use `KEEP` to force the linker to insert all input sections named
`.vector_table.reset_vector` right after the initial SP value. The only symbol located in that
section is `RESET_VECTOR`, so this will effectively place `RESET_VECTOR` second in the vector table.

The output `.text` section contains:

``` text
{{#include ../ci/memory-layout/link.x:27}}
```

This includes all the input sections named `.text` and `.text.*`. Note that we don't use `KEEP`
here to let the linker discard unused sections.

Finally, we use the special `/DISCARD/` section to discard

``` text
{{#include ../ci/memory-layout/link.x:32}}
```

input sections named `.ARM.exidx.*`. These sections are related to exception handling but we are not
doing stack unwinding on panics and they take up space in Flash memory, so we just discard them.

## 全放到一起去

Now we can link the application. For reference, here's the complete Rust program:

``` rust
{{#include ../ci/memory-layout/src/main.rs}}
```

We have to tweak the linker process to make it use our linker script. This is done
passing the `-C link-arg` flag to `rustc`. This can be done with `cargo-rustc` or
`cargo-build`.

**IMPORTANT**: Make sure you have the `.cargo/config` file that was added at the
end of the last section before running this command.

Using the `cargo-rustc` subcommand:

``` console
$ cargo rustc -- -C link-arg=-Tlink.x
```

Or you can set the rustflags in `.cargo/config` and continue using the
`cargo-build` subcommand. We'll do the latter because it better integrates with
`cargo-binutils`.

``` console
# modify .cargo/config so it has these contents
$ cat .cargo/config
```

``` toml
{{#include ../ci/memory-layout/.cargo/config}}
```

The `[target.thumbv7m-none-eabi]` part says that these flags will only be used
when cross compiling to that target.

## 检查它

现在让我们检查下输出的二进制项，确保存储布局跟我们想要的一样
(这需要 [`cargo-binutils`](https://github.com/rust-embedded/cargo-binutils#readme)):

``` console
$ cargo objdump --bin app -- -d --no-show-raw-insn
```

``` text
{{#include ../ci/memory-layout/app.text.objdump}}
```

这是 `.text` section的反汇编。我们看到重置处理函数，名为 `Reset`，位于 `0x8` 地址。

``` console
$ cargo objdump --bin app -- -s --section .vector_table
```

``` text
{{#include ../ci/memory-layout/app.vector_table.objdump}}
```

这是 `.vector_table` section 的内容。我们可以看到section开始于地址 `0x0` 且 section 的第一个字是 `0x2001_0000` (`objdump` 的输出是小端模式)。这是初始的SP值，它与RAM的末尾地址匹配。第二个字是 `0x9`；这是重置处理函数的 *thumb mode* 地址。当一个函数运行在thumb mode下，它的地址的第一位被设置成1 。

## 测试它

这个程序是一个有效的LM3S6965程序；我们可以在一个虚拟微控制器(QEMU)中执行它去测试。

``` console
$ # 这个程序将会阻塞住
$ qemu-system-arm \
      -cpu cortex-m3 \
      -machine lm3s6965evb \
      -gdb tcp::3333 \
      -S \
      -nographic \
      -kernel target/thumbv7m-none-eabi/debug/app
```

``` console
$ # 在一个不同的终端上
$ arm-none-eabi-gdb -q target/thumbv7m-none-eabi/debug/app
Reading symbols from target/thumbv7m-none-eabi/debug/app...done.

(gdb) target remote :3333
Remote debugging using :3333
Reset () at src/main.rs:8
8       pub unsafe extern "C" fn Reset() -> ! {

(gdb) # the SP has the initial value we programmed in the vector table
(gdb) print/x $sp
$1 = 0x20010000

(gdb) step
9           let _x = 42;

(gdb) step
12          loop {}

(gdb) # next we inspect the stack variable `_x`
(gdb) print _x
$2 = 42

(gdb) print &_x
$3 = (i32 *) 0x2000fffc

(gdb) quit
```
