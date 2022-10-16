# 直接存储器访问 (DMA)

本节涉及到围绕DMA传输搭建一个存储安全的API的核心需求。

DMA外设被用来以并行于处理器的工作(主程序的执行)的方式来执行存储传输。一个DMA传输或多或少等于启动一个进程(看[`thread::spawn`])去执行一个`memcpy` 。我们将用fork-join模型去解释一个存储安全的API的要求。

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

一开始我们将使用一个存储不安全的API，然后我们将迭代它直到它完全变成存储安全的API。在每一步，我们都将向你展示如何打破API，
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

在`start`中我们启动了一个DMA传输以填充一个在堆上分配的数组，然后`mem::forget`被返回的`Transfer`值。然后我们继续从`start`返回并执行函数`bar` 。

这一系列操作导致了未定义的行为。DMA传输向栈的存储区写入，但是当`start`返回时，那块存储区域会被释放，
然后被`bar`重新用来分配像是`x`和`y`这样的变量。在运行时，这可能会导致变量`x`和`y`随机更改其值。DMA传输
也会覆盖掉被函数`bar`的序言推入栈中的状态(比如link寄存器)。

注意如果我们不用`mem::forget`，而是`mem::drop`，
Note that if we had not use `mem::forget`, but `mem::drop`, it would have been
possible to make `Transfer`'s destructor stop the DMA transfer and then the
program would have been safe. But one can *not* rely on destructors running to
enforce memory safety because `mem::forget` and memory leaks (see RC cycles) are
safe in Rust.

We can fix this particular problem by changing the lifetime of the buffer from
`'a` to `'static` in both APIs.

``` rust
{{#include ../ci/dma/examples/two.rs:7:12}}
{{#include ../ci/dma/examples/two.rs:21:27}}
{{#include ../ci/dma/examples/two.rs:35:36}}
```

If we try to replicate the previous problem we note that `mem::forget` no longer
causes problems.

``` rust
{{#include ../ci/dma/examples/two.rs:40:52}}
{{#include ../ci/dma/examples/two.rs:54:61}}
```

As before, the DMA transfer continues after `mem::forget`-ing the `Transfer`
value. This time that's not an issue because `buf` is statically allocated
(e.g. `static mut` variable) and not on the stack.

## Overlapping use

Our API doesn't prevent the user from using the `Serial` interface while the DMA
transfer is in progress. This could lead the transfer to fail or data to be
lost.

There are several ways to prevent overlapping use. One way is to have `Transfer`
take ownership of `Serial1` and return it back when `wait` is called.

``` rust
{{#include ../ci/dma/examples/three.rs:7:32}}
{{#include ../ci/dma/examples/three.rs:40:53}}
{{#include ../ci/dma/examples/three.rs:60:68}}
```
The move semantics statically prevent access to `Serial1` while the transfer is
in progress.

``` rust
{{#include ../ci/dma/examples/three.rs:71:81}}
```

There are other ways to prevent overlapping use. For example, a (`Cell`) flag
that indicates whether a DMA transfer is in progress could be added to
`Serial1`. When the flag is set `read`, `write`, `read_exact` and `write_all`
would all return an error (e.g. `Error::InUse`) at runtime. The flag would be
set when `write_all` / `read_exact` is used and cleared in `Transfer.wait`.

## Compiler (mis)optimizations

The compiler is free to re-order and merge non-volatile memory operations to
better optimize a program. With our current API, this freedom can lead to
undefined behavior. Consider the following example:

``` rust
{{#include ../ci/dma/examples/three.rs:84:97}}
```

Here the compiler is free to move `buf.reverse()` before `t.wait()`, which would
result in a data race: both the processor and the DMA would end up modifying
`buf` at the same time. Similarly the compiler can move the zeroing operation to
after `read_exact`, which would also result in a data race.

To prevent these problematic reorderings we can use a [`compiler_fence`]

[`compiler_fence`]: https://doc.rust-lang.org/core/sync/atomic/fn.compiler_fence.html

``` rust
{{#include ../ci/dma/examples/four.rs:9:65}}
```

We use `Ordering::Release` in `read_exact` and `write_all` to prevent all
preceding memory operations from being moved *after* `self.dma.start()`, which
performs a volatile write.

Likewise, we use `Ordering::Acquire` in `Transfer.wait` to prevent all
subsequent memory operations from being moved *before* `self.is_done()`, which
performs a volatile read.

To better visualize the effect of the fences here's a slightly tweaked version
of the example from the previous section. We have added the fences and their
orderings in the comments.

``` rust
{{#include ../ci/dma/examples/four.rs:68:87}}
```

The zeroing operation can *not* be moved *after* `read_exact` due to the
`Release` fence. Similarly, the `reverse` operation can *not* be moved *before*
`wait` due to the `Acquire` fence. The memory operations *between* both fences
*can* be freely reordered across the fences but none of those operations
involves `buf` so such reorderings do *not* result in undefined behavior.

Note that `compiler_fence` is a bit stronger than what's required. For example,
the fences will prevent the operations on `x` from being merged even though we
know that `buf` doesn't overlap with `x` (due to Rust aliasing rules). However,
there exist no intrinsic that's more fine grained than `compiler_fence`.

### 我们不需要内存屏障吗？

That depends on the target architecture. In the case of Cortex M0 to M4F cores,
[AN321] says:

[AN321]: https://static.docs.arm.com/dai0321/a/DAI0321A_programming_guide_memory_barriers_for_m_profile.pdf

> 3.2 Typical usages
>
> (..)
>
> The use of DMB is rarely needed in Cortex-M processors because they do not
> reorder memory transactions. However, it is needed if the software is to be
> reused on other ARM processors, especially multi-master systems. For example:
>
> - DMA controller configuration. A barrier is required between a CPU memory
>   access and a DMA operation.
>
> (..)
>
> 4.18 Multi-master systems
>
> (..)
>
> Omitting the DMB or DSB instruction in the examples in Figure 41 on page 47
> and Figure 42 would not cause any error because the Cortex-M processors:
>
> - do not re-order memory transfers
> - do not permit two write transfers to be overlapped.

Where Figure 41 shows a DMB (memory barrier) instruction being used before
starting a DMA transaction.

In the case of Cortex-M7 cores you'll need memory barriers (DMB/DSB) if you are
using the data cache (DCache), unless you manually invalidate the buffer used by
the DMA. Even with the data cache disabled, memory barriers might still be
required to avoid reordering in the store buffer.

If your target is a multi-core system then it's very likely that you'll need
memory barriers.

If you do need the memory barrier then you need to use [`atomic::fence`] instead
of `compiler_fence`. That should generate a DMB instruction on Cortex-M devices.

[`atomic::fence`]: https://doc.rust-lang.org/core/sync/atomic/fn.fence.html

## Generic buffer

Our API is more restrictive that it needs to be. For example, the following
program won't be accepted even though it's valid.

``` rust
{{#include ../ci/dma/examples/five.rs:67:85}}
```

To accept such program we can make the buffer argument generic.

``` rust
{{#include ../ci/dma/examples/five.rs:9:65}}
```

> **NOTE:** `AsRef<[u8]>` (`AsMut<[u8]>`) could have been used instead of
> `AsSlice<Element = u8>` (`AsMutSlice<Element = u8`).

Now the `reuse` program will be accepted.

## Immovable buffers

With this modification the API will also accept arrays by value (e.g. `[u8;
16]`). However, using arrays can result in pointer invalidation. Consider the
following program.

``` rust
{{#include ../ci/dma/examples/five.rs:88:103}}
{{#include ../ci/dma/examples/five.rs:105:112}}
```

The `read_exact` operation will use the address of the `buffer` local to the
`start` function. That local `buffer` will be freed when `start` returns and the
pointer used in `read_exact` will become invalidated. You'll end up with a
situation similar to the [`unsound`](#dealing-with-memforget) example.

To avoid this problem we require that the buffer used with our API retains its
memory location even when it's moved. The [`Pin`] newtype provides such
guarantee. We can update our API to required that all buffers are "pinned"
first.

[`Pin`]: https://doc.rust-lang.org/nightly/std/pin/index.html

> **NOTE:** To compile all the programs below this point you'll need Rust
> `>=1.33.0`. As of time of writing (2019-01-04) that means using the nightly
> channel.

``` rust
{{#include ../ci/dma/examples/six.rs:16:33}}
{{#include ../ci/dma/examples/six.rs:48:59}}
{{#include ../ci/dma/examples/six.rs:74:75}}
```

> **NOTE:** We could have used the [`StableDeref`] trait instead of the `Pin`
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

Now the problematic program will be rejected.

## Destructors

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
