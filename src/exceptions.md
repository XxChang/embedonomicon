# 异常处理

在"存储布局"部分，我们刚开始的时候决定说得简单点，所以忽略了对异常的处理。在这部分，我们将添加对异常处理的支持；这将作为一个如何在稳定版的Rust中(比如 不依赖不稳定的 `#[linkage = "weak"]` 属性，它让一个符号变成弱链接)实现编译时可重载行为的案例。

## 背景信息

简而言之，*异常*是Cortex-M和其它架构为了让应用可以响应异步的，通常是外部的，事件，所提供的一种机制。经典的(硬件)中断是大多数人所知道的最常见的异常类型。

Cortex-M异常机制工作起来像下面一样:
当处理器接收到与某个异常的类型相关的信号或者事件，它挂起现在的子程序的执行(通过将状态存进调用栈中)然后在一个新的栈帧中继续执行相关的异常处理函数，也就是另一个子程序。在异常处理函数执行完成之后(比如 从它返回后)，处理器恢复被挂起的子程序的执行。

处理器使用向量表去确定执行哪个处理函数。表中的每一项包含一个指向一个处理函数的指针，每一项关联的异常类型都不一样。比如，第二项是重置处理函数，第三项是NMI(不可屏蔽中断)处理函数，等等。

像之前提到的，处理器希望向量表在存储中的某个特定的位置，在运行时处理器可能用到表中的每一项。因此，表中的各项必须包含有效的值。此外，我们希望`rt` crate是灵活的，因此终端用户可以定制每个异常处理函数的行为。最后，向量表要坐落在只读存储中，或者在不容易被修改的存储中，因此用户必须静态地注册处理函数，而不是在运行时。

为了满足所有的这些需求，我们将给`rt` crate中的向量表所有的项分配一个*默认值*，但是让这些值变*弱*以让终端用户可以在编译时重载它们。

## Rust部分

让我们看下所有的这些要如何被实现。为了方便，我们将只使用向量表的前16个项；这些项不是特定于设备的，所以它们在所有类型的Cotex-M微控制器上都有相同的作用。

我们做的第一件事是在`rt` create的代码中创造一个向量(指向异常处理函数的指针)数组:

``` console
$ sed -n 56,91p ../rt/src/lib.rs
```

``` rust
{{#include ../ci/exceptions/rt/src/lib.rs:56:91}}
```

向量表中的一些项是*保留的*；ARM文档说它们应该被分配成 `0` 值，所以我们使用一个联合体来完成它。需要指向一个处理函数的项必须使用*external*函数；这很重要，因为它可以让终端用户来*提供*实际的函数定义。

接下来，我们在Rust代码中定义一个默认的异常处理函数。没有被终端用户分配的异常将使用这个默认处理函数。

``` console
$ tail -n4 ../rt/src/lib.rs
```

``` rust
{{#include ../ci/exceptions/rt/src/lib.rs:93:97}}
```

## 链接器脚本部分

在链接器脚本那部分，我们将这些新的异常向量放在重置向量之后。

``` console
$ sed -n 12,25p ../rt/link.x
```

``` text
{{#include ../ci/exceptions/rt/link.x:12:27}}
```

并且我们使用 `PROVIDE` 给我们在`rt`中未定义的处理函数赋予一个默认值 (`NMI`和上面的其它处理函数):

``` console
$ tail -n8 ../rt/link.x
```

``` text
PROVIDE(NMI = DefaultExceptionHandler);
PROVIDE(HardFault = DefaultExceptionHandler);
PROVIDE(MemManage = DefaultExceptionHandler);
PROVIDE(BusFault = DefaultExceptionHandler);
PROVIDE(UsageFault = DefaultExceptionHandler);
PROVIDE(SVCall = DefaultExceptionHandler);
PROVIDE(PendSV = DefaultExceptionHandler);
PROVIDE(SysTick = DefaultExceptionHandler);
```

当检测完所有的输入目标文件而等号左侧的符号仍然没有定义的时候，`PROVIDE` 才会发挥作用。也就是用户没有为相关的异常实现处理函数时。

## 测试它

这就完了！`rt` crate现在支持异常处理函数了。我们可以用下列的应用测试它:

> **注意**: 在QEMU中生成一个异常很难。在实际的硬件上对一个无效的存储地址进行
> 一个读取是足够产生一个中断的，但是QEMU却能接受这个操作并且返回零。一个陷入指
> 令在QEMU和硬件上都可以发挥中断的作用，但是不幸的是在稳定版上不可以用它，所以你需要
> 暂时切换到nightly版本中去运行这个和下个案例。

``` rust
{{#include ../ci/exceptions/app/src/main.rs}}
```

``` console
(gdb) target remote :3333
Remote debugging using :3333
Reset () at ../rt/src/lib.rs:7
7       pub unsafe extern "C" fn Reset() -> ! {

(gdb) b DefaultExceptionHandler
Breakpoint 1 at 0xec: file ../rt/src/lib.rs, line 95.

(gdb) continue
Continuing.

Breakpoint 1, DefaultExceptionHandler ()
    at ../rt/src/lib.rs:95
95          loop {}

(gdb) list
90          Vector { handler: SysTick },
91      ];
92
93      #[no_mangle]
94      pub extern "C" fn DefaultExceptionHandler() {
95          loop {}
96      }
```

为了完整性，这里列出程序被优化过的版本的反汇编:

``` console
$ cargo objdump --bin app --release -- -d --no-show-raw-insn --print-imm-hex
```

``` text
{{#include ../ci/exceptions/app/app.objdump:1:30}}
```

``` console
$ cargo objdump --bin app --release -- -s -j .vector_table
```

``` text
{{#include ../ci/exceptions/app/app.vector_table.objdump}}
```

向量表现在像是集合了这本书中迄今为止所有的代码片段。总结下:
- 在早期的存储章节的[_检查它_]部分，我们知道了:
    - 向量表中第一项包含了栈指针的初始值。
    - Objdump使用`小端`格式打印，所以栈开始于 `0x2001_0000` 。
    - 第二项指向地址 `0x0000_0045`，重置处理函数。
        - 在上面的反汇编中可以看到重置处理函数的地址，是 `0x44` 。
        - 由于对齐的要求被设置成1的第一位不会改变地址。而是，它让函数在 _thumb mode_ 下执行。
- 之后，可以看到在`0x83`和`0x00`之间交替的地址模式。
    - 看下上面的反汇编，很明显 `0x83` 指的是 `DefaultExceptionHandler` (`0x84`用thumb模式执行)。
    - 在这个章节早期所设置的向量表和[Cortex-M的向量表布局]的模式之间来回查看，很明显每次有个带处理函数的项出现在表中，`DefaultExceptionHandler`的地址就会出现。
    - 可以看到Rust代码中的向量表的数据结构的布局与Cortex-M向量表中的保留项依次对齐了。因此，所有的保留项被正确的设置成了零值。

[_检查它_]: https://xxchang.github.io/embedonomicon/memory-layout.html#inspecting-it
[Cortex-M的向量表布局]: https://developer.arm.com/docs/dui0552/latest/the-cortex-m3-processor/exception-model/vector-table

## 重载一个处理函数

为了重载一个异常处理函数，用户必须提供一个函数，其符号名完全匹配我们在`EXCEPTIONS`中使用的名字。

``` rust
{{#include ../ci/exceptions/app2/src/main.rs}}
```

你可以在QEMU中测试它

``` console
(gdb) target remote :3333
Remote debugging using :3333
Reset () at /home/japaric/rust/embedonomicon/ci/exceptions/rt/src/lib.rs:7
7       pub unsafe extern "C" fn Reset() -> ! {

(gdb) b HardFault
Breakpoint 1 at 0x44: file src/main.rs, line 18.

(gdb) continue
Continuing.

Breakpoint 1, HardFault () at src/main.rs:18
18          loop {}

(gdb) list
13      }
14
15      #[no_mangle]
16      pub extern "C" fn HardFault() -> ! {
17          // do something interesting here
18          loop {}
19      }
```

程序现在执行了用户定义的`HardFault`函数而不是`rt` crate中的`DefaultExceptionHandler` 。

与我们在`main`接口中进行的第一次尝试一样，这个实现的问题是没有类型安全性。它也容易混淆异常的名字，但是不会生成一个错误或者警告。而仅仅是忽略用户定义的处理函数。这些问题可以使用一个像是在`cortex-m-rt` v0.5.x 中定义的[`exception!`]宏和`cortex-m-rt` v0.6.x中的[`exception`]属性来解决。

[`exception!`]: https://github.com/japaric/cortex-m-rt/blob/v0.5.1/src/lib.rs#L792
[`exception`]: https://github.com/rust-embedded/cortex-m-rt/blob/v0.6.3/macros/src/lib.rs#L254
