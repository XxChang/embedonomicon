# 直接存储器访问 (DMA)

本节会围绕DMA传输，讨论要搭建一个内存安全的API的核心需求。

DMA外设被用来以并行于处理器的工作(主程序的执行)的方式来执行存储传输。一个DMA传输或多或少等于启动一个进程(看[`thread::spawn`])去执行一个`memcpy` 。我们将用fork-join模型去解释一个内存安全的API的要求。

[`thread::spawn`]: https://doc.rust-lang.org/std/thread/fn.spawn.html

考虑下面的DMA数据类型：

``` rust
{{#include ../ci/dma/src/lib.rs:6:57}}
{{#include ../ci/dma/src/lib.rs:59:60}}
```

假设`Dma1Channel1`被静态地配置成按one-shot的模式(也即不是circular模式)使用串口(又称作UART或者USART) #1，`Serial1`。
`Serial1`提供下面的*阻塞*版的API：

``` rust
{{#include ../ci/dma/src/lib.rs:62:72}}
{{#include ../ci/dma/src/lib.rs:74:80}}
{{#include ../ci/dma/src/lib.rs:82:83}}
```

假设我们想要将`Serial1` API扩展成可以(a)异步地发送一个缓存区和(b)异步地填充一个缓存区。

一开始我们将使用一个存储不安全的API，然后我们将迭代它直到它完全变成存储安全的API。在每一步，我们都将向你展示如何破开API，
让你意识到当使用异步的存储操作时，有哪些问题需要被解决。

## 开场

作为开端，让我们尝试使用[`Write::write_all`] API作为参考。为了简便，让我们忽略所有的错误处理。

[`Write::write_all`]: https://doc.rust-lang.org/std/io/trait.Write.html#method.write_all

``` rust
{{#include ../ci/dma/examples/one.rs:7:47}}
```

> **注意：** 不用像上面的API一样，`Transfer`的API也可以暴露一个futures或者generator。
> 这是一个API设计问题，与整个API的内存安全性关系不大，因此我们在本文中不会深入讨论。

我们也可以实现一个异步版本的[`Read::read_exact`] 。

[`Read::read_exact`]: https://doc.rust-lang.org/std/io/trait.Read.html#method.read_exact

``` rust
{{#include ../ci/dma/examples/one.rs:49:63}}
```

这里是`write_all` API的用法：

``` rust
{{#include ../ci/dma/examples/one.rs:66:71}}
```

这是使用`read_exact` API的一个例子：

``` rust
{{#include ../ci/dma/examples/one.rs:74:86}}
```

## `mem::forget`

[`mem::forget`]是一个安全的API。如果我们的API真的是安全的，那么我们应该能够将两者结合使用而不会出现未定义的行为。然而，情况并非如此；考虑下下面的例子：

[`mem::forget`]: https://doc.rust-lang.org/std/mem/fn.forget.html

``` rust
{{#include ../ci/dma/examples/one.rs:91:103}}
{{#include ../ci/dma/examples/one.rs:105:112}}
```

在`start`中我们启动了一个DMA传输以填充一个在堆上分配的数组，然后`mem::forget`了被返回的`Transfer`值。然后我们继续从`start`返回并执行函数`bar` 。

这一系列操作导致了未定义的行为。DMA传输向栈的存储区写入，但是当`start`返回时，那块存储区域会被释放，
然后被`bar`重新用来分配像是`x`和`y`这样的变量。在运行时，这可能会导致变量`x`和`y`随机更改其值。DMA传输
也会覆盖掉被函数`bar`的序言推入栈中的状态(比如link寄存器)。

注意如果我们不用`mem::forget`，而是`mem::drop`，可以让`Transfer`的析构函数停止DMA的传输，这样程序就变成了安全的了。但是*不*能依赖于运行析构函数来加强存储安全性因为`mem::forget`和内存泄露(看下RC cycles)在Rust中是安全的。

通过在APIs中把缓存的生命周期从`'a`变成`'static`来修复这个问题。

``` rust
{{#include ../ci/dma/examples/two.rs:7:12}}
{{#include ../ci/dma/examples/two.rs:21:27}}
{{#include ../ci/dma/examples/two.rs:35:36}}
```

如果我们尝试复现先前的问题，我们注意到`mem::forget`不再引起问题了。

``` rust
{{#include ../ci/dma/examples/two.rs:40:52}}
{{#include ../ci/dma/examples/two.rs:54:61}}
```

像之前一样，在`mem::forget` `Transfer`的值之后，DMA传输继续运行着。这次没有问题了，因为`buf`是静态分配的(比如`static mut`变量)，不是在栈上。

## 重复使用(Overlapping use)

我们的API没有阻止用户在DMA传输过程中再次使用`Serial`接口。这可能导致传输失败或者数据丢失。

有许多方法可以禁止重叠使用。一个方法是让`Transfer`获取`Serial1`的所有权，然后当`wait`被调用时将它返回。

``` rust
{{#include ../ci/dma/examples/three.rs:7:32}}
{{#include ../ci/dma/examples/three.rs:40:53}}
{{#include ../ci/dma/examples/three.rs:60:68}}
```
移动语义静态地阻止了当传输在进行时对`Serial1`的访问。

``` rust
{{#include ../ci/dma/examples/three.rs:71:81}}
```

还有其它方法可以防止重叠使用。比如，可以往`Serial1`添加一个(`Cell`)标志，其指出是否一个DMA传输正在进行中。
当标志被设置了，`read`，`write`，`read_exact`和`write_all`全都会在运行时返回一个错误(比如`Error::InUse`)。
当使用`write_all` / `read_exact`时，会设置标志，在`Transfer.wait`中，标志会被清除。

## 编译器(误)优化

编译器可以自由地重新排序和合并不是volatile的存储操作以更好地优化一个程序。使用我们现在的API，这种自由度会导致未定义的行为。想一下下面的例子：

``` rust
{{#include ../ci/dma/examples/three.rs:84:97}}
```

这里编译器可以将`buf.reverse()`移到`t.wait()`之前，其将导致一个数据竞争问题：处理器和DMA最终都会同时修改`buf` 。同样地编译器可以将赋零操作放到`read_exact`之后，它也会导致一个数据竞争问题。

为了避免这些存在问题的重排序，我们可以使用一个 [`compiler_fence`]

[`compiler_fence`]: https://doc.rust-lang.org/core/sync/atomic/fn.compiler_fence.html

``` rust
{{#include ../ci/dma/examples/four.rs:9:65}}
```

我们在`read_exact`和`write_all`中使用`Ordering::Release`以避免所有的正在进行中的存储操作被移动到`self.dma.start()`后面去，其执行了一个volatile写入。

同样地，我们在`Transfer.wait`中使用`Ordering::Acquire`以避免所有的后续的存储操作被移到`self.is_done()`*之前*，其执行了一个volatile读入。

为了更好地展示fences的影响，稍微修改下上个部分中的例子。我们将fences和它们的orderings添加到注释中。

``` rust
{{#include ../ci/dma/examples/four.rs:68:87}}
```

由于`Release` fence，赋零操作*不*能被移到`read_exact`*之后*。同样地，由于`Acquire` fence，`reverse`操作*不*能被移动`wait`之前。
在两个fences*之间*的存储操作*可以*在fences间自由地重新排序，但是这些操作都不会涉及到`buf`，所以这种重新排序*不*会导致未定义的行为。

请注意`compiler_fence`比要求的强一些。比如，fences将防止在`x`上的操作被合并即使我们知道`buf`不会与`x`重叠(由于Rust的别名规则)。
然而，没有比`compiler_fence`更精细的内部函数了。

### 我们需不需要内存屏障？

这取决于目标平台的架构。在Cortex M0到M4F核心的例子里，[AN321]说到：

[AN321]: https://static.docs.arm.com/dai0321/a/DAI0321A_programming_guide_memory_barriers_for_m_profile.pdf

> 3.2 主要场景
>
> (..)
>
> 在Cortex-M处理器中，很少需要用到DMB因为它们不会重新排序存储传输。
> 然而，如果软件要在其它ARM处理器中复用，那么就需要用到，特别是多主机系统。比如：
>
> - DMA控制器配置。在CPU存储访问和一个DMA操作间需要一个屏障。
>
> (..)
>
> 4.18 多主机系统
>
> (..)
>
> 把47页图41和图42的例子中的DMB或者DSB指令去除掉不会导致任何错误，因为Cortex-M处理器：
>
> - 不会重新排序存储传输。
> - 不会允许两个写传输重叠。

这里图41中展示了在启动DMA传输前使用了一个DMB（存储屏障）指令。

在Cortex-M7内核的例子中，如果你使用了数据缓存（DCache），那么你需要存储屏障（DMB/DSB），除非你手动地无效化被DMA使用的缓存。即使将数据缓存取消掉，可能依然需要内存屏障以避免存储缓存中出现重新排序。

如果你的目标平台是一个多核系统，那么很可能你需要内存屏障。

如果你需要内存屏障，那么你需要使用[`atomic::fence`]而不是`compiler_fence`。这在Cortex-M设备上会生成一个DMB指令。

[`atomic::fence`]: https://doc.rust-lang.org/core/sync/atomic/fn.fence.html

## 泛化缓存

我们的API太受限了。比如，下面的程序即使是有效的也不会被通过。

``` rust
{{#include ../ci/dma/examples/five.rs:67:85}}
```

为了能接受这样的程序，我们可以让缓存参数更泛化点。

``` rust
{{#include ../ci/dma/examples/five.rs:9:65}}
```

> **注意:** 可以使用 `AsRef<[u8]>` (`AsMut<[u8]>`) 而不是
> `AsSlice<Element = u8>` (`AsMutSlice<Element = u8`).

现在 `reuse` 程序可以通过了。

## 不可移动的缓存

这么修改后，API也可以通过值传递接受数组。（比如 `[u8;
16]`）。然后，使用数组会导致指针无效化。考虑下面的程序。
``` rust
{{#include ../ci/dma/examples/five.rs:88:103}}
{{#include ../ci/dma/examples/five.rs:105:112}}
```

`read_exact` 操作将使用位于 `start` 函数的 `buffer` 的地址。当 `start` 返回时，局部的 `buffer` 将会被释放，在 `read_exact` 中使用的指针将会变得无效化。你最后会遇到与 [`unsound`](#dealing-with-memforget) 案例中一样的情况。

为了避免这个问题，我们要求我们的API使用的缓存即使当它被移动时依然保有它的内存区域。[`Pin`] 类型提供这样的保障。首先我们可以更新我们的API以要求所有的缓存都是 "pinned" 的。

[`Pin`]: https://doc.rust-lang.org/nightly/std/pin/index.html

> **注意：** 要编译下面的所有程序，你的Rust需要 
> `>=1.33.0`。写这本书的时候 (2019-01-04) 这意味着要使用 nightly
> 版的Rust

``` rust
{{#include ../ci/dma/examples/six.rs:16:33}}
{{#include ../ci/dma/examples/six.rs:48:59}}
{{#include ../ci/dma/examples/six.rs:74:75}}
```

> **注意：** 我们可以使用 [`StableDeref`] 特质而不是 `Pin`
> newtype but opted for `Pin` since it's provided in the standard library.

[`StableDeref`]: https://crates.io/crates/stable_deref_trait

With this new API we can use `&'static mut` references, `Box`-ed slices, `Rc`-ed
slices, etc.

``` rust
{{#include ../ci/dma/examples/six.rs:78:89}}
{{#include ../ci/dma/examples/six.rs:91:101}}
```

## `'static` bound

Does pinning let us safely use stack allocated arrays? The answer is *no*.
Consider the following example.

``` rust
{{#include ../ci/dma/examples/six.rs:104:123}}
{{#include ../ci/dma/examples/six.rs:125:132}}
```

As seen many times before, the above program runs into undefined behavior due to
stack frame corruption.

The API is unsound for buffers of type `Pin<&'a mut [u8]>` where `'a` is *not*
`'static`. To prevent the problem we have to add a `'static` bound in some
places.

``` rust
{{#include ../ci/dma/examples/seven.rs:15:25}}
{{#include ../ci/dma/examples/seven.rs:40:51}}
{{#include ../ci/dma/examples/seven.rs:66:67}}
```

现在有问题的程序将被拒绝。

## 析构函数

Now that the API accepts `Box`-es and other types that have destructors we need
to decide what to do when `Transfer` is early-dropped.

Normally, `Transfer` values are consumed using the `wait` method but it's also
possible to, implicitly or explicitly, `drop` the value before the transfer is
over. For example, dropping a `Transfer<Box<[u8]>>` value will cause the buffer
to be deallocated. This can result in undefined behavior if the transfer is
still in progress as the DMA would end up writing to deallocated memory.

In such scenario one option is to make `Transfer.drop` stop the DMA transfer.
The other option is to make `Transfer.drop` wait for the transfer to finish.
We'll pick the former option as it's cheaper.

``` rust
{{#include ../ci/dma/examples/eight.rs:18:72}}
{{#include ../ci/dma/examples/eight.rs:82:99}}
{{#include ../ci/dma/examples/eight.rs:109:117}}
```

Now the DMA transfer will be stopped before the buffer is deallocated.

``` rust
{{#include ../ci/dma/examples/eight.rs:120:134}}
```

## 总结

To sum it up, we need to consider all the following points to achieve  memory
safe DMA transfers:

- Use immovable buffers plus indirection: `Pin<B>`. Alternatively, you can use
  the `StableDeref` trait.

- The ownership of the buffer must be passed to the DMA : `B: 'static`.

- Do *not* rely on destructors running for memory safety. Consider what happens
  if `mem::forget` is used with your API.

- *Do* add a custom destructor that stops the DMA transfer, or waits for it to
  finish. Consider what happens if `mem::drop` is used with your API.

---

This text leaves out up several details required to build a production grade
DMA abstraction, like configuring the DMA channels (e.g. streams, circular vs
one-shot mode, etc.), alignment of buffers, error handling, how to make the
abstraction device-agnostic, etc. All those aspects are left as an exercise for
the reader / community (`:P`).
Overlapping use