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

我们通过使用 `extern "C"` 告诉编译器下面的函数使用C的ABI，而不是Rust的ABI，后者不稳定，将函数变成这里硬件期望的格式。

为了从链接器脚本中指出重置处理函数和重置向量，我们需要它们有一个稳定的符号名因此我们使用 `#[no_mangle]` 。我们需要稍微控制下 `RESET_VECTOR` 的位置，因此我们将它放进一个已知的section中，`.vector_table.reset_vector` 。重置处理函数，`Reset` ，的确切位置不重要。我们只使用编译器默认生成的section 。

这个链接器将忽略有内部链接的符号(也叫内部符号)，遍历输入对象文件的列表，因此我们需要我们的符号具有外部链接。在Rust中让一个符号编程外部的方法只有使它的相关项变成公共的(`pub`)和*可到达的*(在项和crate的根路径间没有私有模块)。

## 链接器脚本部分

一个最小的，将向量表放进正确的位置的链接器脚本如下所示。让我们仔细看看。

``` console
$ cat link.x
```

``` text
{{#include ../ci/memory-layout/link.x}}
```

### `MEMORY`

链接器脚本的这个部分描述了目标中存储区块的大小和位置。有两个存储区块被定义了: `FLASH` 和 `RAM` ；它们都与目标中可用的物理存储关联。这里用的值与LM3S6965微控制器有关。

### `ENTRY`

这里我们指给链接器，符号名为 `Reset` 的重置处理函数是程序的 *entry point* 。链接器会主动丢弃未使用的部分。链接器认为entry point和从它处被调用的函数是*被使用了的*，所以链接器将不会丢弃它们。没有这一行，链接器将会丢弃 `Reset` 函数和所有接下来所有从它处调用的函数。

### `EXTERN`

链接器是懒惰的；一旦它们找到了从entry point处递归引用的所有的符号，它们将停止查看输入对象文件。`EXTERN` 强迫链接器去寻找 `EXTERN` 的参数即使其它所有的被引用的符号都被找到后。因为thumb的一个规则，如果你需要一个符号，其没有被entry point调用，总是出现在输出的二进制项中，你应该使用结合了 `KEEP` 的 `EXTERN` 。

### `SECTIONS`

这部分描述了输入对象文件中的sections(也被称为*input sections*)是如何被安排进输出对象文件的sections(也被称为output sections)中的或者它们是否应该被丢弃。这里，我们定义两个输出sections:

``` text
  .vector_table ORIGIN(FLASH) : { /* .. */ } > FLASH
```

`.vector_table` 包含向量表且坐落于 `FLASH` 存储的开始处。

``` text
  .text : { /* .. */ } > FLASH
```

`.text` 包含程序的子例程且坐落于 `FLASH` 的某些位置。它的开始地址没有指定，但是链接器将把它放在先前的输出section，`.vector_table` 之后。

输出 `.vector_table` section包含:

``` text
{{#include ../ci/memory-layout/link.x:18:19}}
```

我们将把(调用)栈放在RAM的末尾(栈是*完全递减的*；它向着更小的地址增长)因此RAM的末尾地址将被用作栈指针(SP)值。使用我们输入的`RAM`存储区块的信息，在链接器中那个地址可以被链接器算出来。

```
{{#include ../ci/memory-layout/link.x:21:22}}
```

接下来，我们使用 `KEEP` 去强迫链接器在初始的SP值之后插入所有的名为 `.vector_table.reset_vector` 的input sections 。位于那个section中的唯一一个符号是 `RESET_VECTOR`，所以这将可以把 `RESEET_VECTOR` 放在向量表的第二个位置。

输出的 `.text` section 包含:

``` text
{{#include ../ci/memory-layout/link.x:27}}
```

这包括所有的名为 `.text` 和 `.text.*` 的input sections 。注意我们这里没有使用 `KEEP` 去让链接器丢弃不使用的部分。

最后，我们使用特殊的 `/DISCARD/` section 去丢弃

``` text
{{#include ../ci/memory-layout/link.x:32}}
```

名为 `.ARM.exidx.*` 的input sections 。这些sections与异常处理有关，但是我们在恐慌时不进行栈展开，它们占用Flash的存储空间，因此我们就丢弃它们。

## 全放到一起去

现在我们能链接应用了。作为参考，这里是完整的Rust程序:

``` rust
{{#include ../ci/memory-layout/src/main.rs}}
```

我们不得不修改链接的过程让它使用我们的链接器脚本。通过传递 `-C link-arg` 标志给 `rustc` 来完成它。使用 `cargo-rustc` 或者 `cargo-build` 就可以完成了。

**重要**: 在运行这个命令之前确保你有 `.cargo/config` 文件，其在上个章节末尾处被添加。

使用 `cargo-rustc` 子命令:

``` console
$ cargo rustc -- -C link-arg=-Tlink.x
```

或者你可以在 `.cargo/config` 中设置rustflags，其继续使用 `cargo-build` 子命令。我们将会使用后者因为它与`cargo-binutils`集成的更好。

``` console
# 将 .cargo/config 修改成这些内容
$ cat .cargo/config
```

``` toml
{{#include ../ci/memory-layout/.cargo/config}}
```

`[target.thumbv7m-none-eabi]` 部分告知这些标志只会被用于当交叉编译的目标是`thumbv7m-none-eabi`时。

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
