# 异常处理

在"存储布局"部分，我们决定从简单处开始并忽略异常的处理。在这部分，我们将添加与处理它们有关的支持；这作为如何在稳定版的Rust中(比如 不依赖不稳定的 `#[linkage = "weak"]` 属性，它让一个符号变成弱链接)实现编译时可重载的行为的示例。

## 背景信息

简而言之，*异常*是Cortex-M和其它架构提供的一个机制，为了让应用可以响应异步，通常是外部的，事件。异常最常见的类型，大多数人将知道的，是经典的(硬件)中断。

Cortex-M异常机制工作起来像这样:
当处理器接收到与某个异常的类型相关的信号或者事件，它挂起现在的子例程的执行(通过将状态存进调用栈中)然后在一个新的栈帧中继续执行相关的异常处理函数，另一个子例程。在异常处理函数执行完成之后(比如 从它返回)，处理器恢复被挂起的子例程的执行。

处理器使用向量表去确定执行哪个处理函数。表中的每一项包含一个指向一个处理函数的指针，每一项与一个不同的异常类型关联起来。比如，第二项是重置处理函数，第三项是NMI(不可屏蔽中断)处理函数，等等。

像之前提到的，处理器希望向量表在存储中某个特定的位置，在运行时处理器可能用到表中的每一项。因此，表中的各项必须包含有效的值。此外，我们希望`rt` crate是灵活的，因此终端用户可以定制每个异常处理函数的行为。最后，向量表要坐落在只读存储中，或者有些在不那么容易被修改的存储中，因此用户必须静态地注册处理函数，而不是在运行时。

To satisfy all these constraints, we'll assign a *default* value to all the entries of the vector
table in the `rt` crate, but make these values kind of *weak* to let the end user override them
at compile time.

## Rust side

Let's see how all this can be implemented. For simplicity, we'll only work with the first 16 entries
of the vector table; these entries are not device specific so they have the same function on any
kind of Cortex-M microcontroller.

The first thing we'll do is create an array of vectors (pointers to exception handlers) in the
`rt` crate's code:

``` console
$ sed -n 56,91p ../rt/src/lib.rs
```

``` rust
{{#include ../ci/exceptions/rt/src/lib.rs:56:91}}
```

Some of the entries in the vector table are *reserved*; the ARM documentation states that they
should be assigned the value `0` so we use a union to do exactly that. The entries that must point
to a handler make use of *external* functions; this is important because it lets the end user
*provide* the actual function definition.

Next, we define a default exception handler in the Rust code. Exceptions that have not been assigned
a handler by the end user will make use of this default handler.

``` console
$ tail -n4 ../rt/src/lib.rs
```

``` rust
{{#include ../ci/exceptions/rt/src/lib.rs:93:97}}
```

## Linker script side

On the linker script side, we place these new exception vectors right after the reset vector.

``` console
$ sed -n 12,25p ../rt/link.x
```

``` text
{{#include ../ci/exceptions/rt/link.x:12:27}}
```

And we use `PROVIDE` to give a default value to the handlers that we left undefined in `rt` (`NMI`
and the others above):

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

`PROVIDE` only takes effect when the symbol to the left of the equal sign is still undefined after
inspecting all the input object files. This is the scenario where the user didn't implement the
handler for the respective exception.

## Testing it

That's it! The `rt` crate now has support for exception handlers. We can test it out with following
application:

> **NOTE**: Turns out it's hard to generate an exception in QEMU. On real
> hardware a read to an invalid memory address (i.e. outside of the Flash and
> RAM regions) would be enough but QEMU happily accepts the operation and
> returns zero. A trap instruction works on both QEMU and hardware but
> unfortunately it's not available on stable so you'll have to temporarily
> switch to nightly to run this and the next example.

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

And for completeness, here's the disassembly of the optimized version of the program:

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

The vector table now resembles the results of all the code snippets in this book
  so far. To summarize:
- In the [_Inspecting it_] section of the earlier memory chapter, we learned
  that:
    - The first entry in the vector table contains the initial value of the
      stack pointer.
    - Objdump prints in `little endian` format, so the stack starts at
      `0x2001_0000`.
    - The second entry points to address `0x0000_0045`, the Reset handler.
        - The address of the Reset handler can be seen in the disassembly above,
          being `0x44`.
        - The first bit being set to 1 does not alter the address due to
          alignment requirements. Instead, it causes the function to be executed
          in _thumb mode_.
- Afterwards, a pattern of addresses alternating between `0x83` and `0x00` is
  visible.
    - Looking at the disassembly above, it is clear that `0x83` refers to the
      `DefaultExceptionHandler` (`0x84` executed in thumb mode).
    - Cross referencing the pattern to the vector table that was set up earlier
      in this chapter (see the definition of `pub static EXCEPTIONS`) with [the
      vector table layout for the Cortex-M], it is clear that the address of the
      `DefaultExceptionHandler` is present each time a respective handler entry
      is present in the table.
    - In turn, it is also visible that the layout of the vector table data
      structure in the Rust code is aligned with all the reserved slots in the
      Cortex-M vector table. Hence, all reserved slots are correctly set to a
      value of zero.

[_Inspecting it_]: https://docs.rust-embedded.org/embedonomicon/memory-layout.html#inspecting-it
[the vector table layout for the Cortex-M]: https://developer.arm.com/docs/dui0552/latest/the-cortex-m3-processor/exception-model/vector-table

## Overriding a handler

To override an exception handler, the user has to provide a function whose symbol name exactly
matches the name we used in `EXCEPTIONS`.

``` rust
{{#include ../ci/exceptions/app2/src/main.rs}}
```

You can test it in QEMU

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

Like our first attempt at a `main` interface, this first implementation has the problem of having no
type safety. It's also easy to mistype the name of the exception, but that doesn't produce an error
or warning. Instead the user defined handler is simply ignored. Those problems can be fixed using a
macro like the [`exception!`] macro defined in `cortex-m-rt` v0.5.x or the
[`exception`] attribute in `cortex-m-rt` v0.6.x.

[`exception!`]: https://github.com/japaric/cortex-m-rt/blob/v0.5.1/src/lib.rs#L792
[`exception`]: https://github.com/rust-embedded/cortex-m-rt/blob/v0.6.3/macros/src/lib.rs#L254
