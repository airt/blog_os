+++
title = "内存分配器设计"
weight = 11
path = "zh-CN/allocator-designs"
date = 2020-01-20

[extra]
chapter = "Memory Management"
# Please update this when updating the translation
translation_based_on_commit = "ac943091147a57fcac8bde8876776c7aaff5c3d8"
# GitHub usernames of the people that translated this post
translators = ["airt"]
+++

本文讲解了如何从头开始实现一个堆分配器。我们会介绍并讨论不同的分配器设计，包括 Bump 分配器、链表分配器和固定大小块分配器。对于每一种设计，我们都将创建一个可用于内核的基本实现。

<!-- more -->

这个系列的 blog 在 [GitHub] 上开放开发。如果你有任何问题或疑问，请在这里开一个 issue 来讨论。你也可以在[底部][at the bottom]发表评论。这篇文章的完整源代码可以在 [`post-11`][post branch] 分支中找到。

[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
<!-- fix for zola anchor checker (target is in template): <a id="comments"> -->
[post branch]: https://github.com/phil-opp/blog_os/tree/post-11

<!-- toc -->

## 简介

在[上一篇文章][previous post]中，我们为内核添加了堆分配的基本支持。为此，我们在页表中[创建了一个新的内存区域][map-heap]，并[使用 `linked_list_allocator` 这个 crate][use-alloc-crate] 来管理这块内存。现在有了一个可用的堆，但我们将大部分工作交给了这个 crate，而没有试图理解它是如何工作的。

[previous post]: @/edition-2/posts/10-heap-allocation/index.md
[map-heap]: @/edition-2/posts/10-heap-allocation/index.md#creating-a-kernel-heap
[use-alloc-crate]: @/edition-2/posts/10-heap-allocation/index.md#using-an-allocator-crate

在本文中，我们将展示如何从头开始创建我们自己的堆分配器，而不是依赖于现有的分配器 crate。我们将讨论不同的分配器设计，包括一个简单的 _Bump 分配器_ 和基本的 _固定大小块分配器_，然后再使用这些知识来实现一个性能更好的分配器 (相较于 `linked_list_allocator` crate)。

### 设计目标

分配器的职责是管理可用的堆内存。它需要在 `alloc` 调用时返回未使用的内存，并跟踪 `dealloc` 释放的内存，以便再次使用。最重要的是，它不能重复分配已使用的内存，这会导致未定义行为。

除了正确性之外，还有许多次要的设计目标。例如，分配器应当高效利用内存，并且保持尽量少的[碎片][_fragmentation_]。此外，它应该适用于并发程序，并可扩展到任意数量的处理器。为了最佳性能，它甚至可以针对 CPU 缓存优化内存布局，以改善[访问局部性][cache locality]并避免[伪共享][false sharing]。

[cache locality]: https://www.geeksforgeeks.org/locality-of-reference-and-cache-operation-in-cache-memory/
[_fragmentation_]: https://en.wikipedia.org/wiki/Fragmentation_(computing)
[false sharing]: https://mechanical-sympathy.blogspot.de/2011/07/false-sharing.html

这些要求会使得好的分配器非常复杂。例如，[jemalloc] 有超过 30,000 行代码。通常我们不希望内核代码这么复杂，因为其中单个错误就可能导致严重的安全漏洞。所幸，相比用户空间代码，内核代码的分配模式通常会简单很多，因此相对简单的分配器设计就足够了。

[jemalloc]: http://jemalloc.net/

下面，我们将介绍三种可能的内核分配器设计，并解释它们的优缺点。

## Bump 分配器

最简单的分配器设计是 _Bump 分配器_ (也称为 _栈分配器_)。它线性分配内存，只记录分配的字节数和分配的次数。它只适用于非常特定的场合，因为他有一个严格的限制: 只能一次性释放所有内存。

### 思路

Bump 分配器背后的思路是通过一个 `next` 变量的增长 (_"bump"_) 来线性分配内存，这个变量总是指向未使用内存的起点。最开始时，`next` 等于堆的起始地址。每次分配时，`next` 会按照分配的大小来增长，使得它总是指向已使用和未使用内存的边界。

![三个时间点的堆内存区域:
 1: 在堆的起始处存在一个分配；`next` 指针指向它的末尾。
 2: 在第一次分配后立即进行第二次分配；`next` 指针指向第二次分配的末尾。
 3: 在第二次分配后立即进行第三次分配；`next` 指针指向第三次分配的末尾。](bump-allocation.svg)

`next` 指针只进行单向移动，因此不会重复分配同一块内存区域。当它到达堆的末尾，无法再分配更多内存，在下一次分配时就会出现内存不足错误。

Bump 分配器通常用计数器实现，每次 `alloc` 调用时加 1，每次 `dealloc` 调用时减 1。当分配计数器达到零时，表示堆上的所有分配已被释放。这时，可以将 `next` 指针重置为堆的起始地址，使得整个堆内存可以再次被分配。

### 实现

我们声明一个新的 `allocator::bump` 子模块来开始：

```rust
// in src/allocator.rs

pub mod bump;
```

子模块的内容位于一个新文件 `src/allocator/bump.rs`，内容如下：

```rust
// in src/allocator/bump.rs

pub struct BumpAllocator {
    heap_start: usize,
    heap_end: usize,
    next: usize,
    allocations: usize,
}

impl BumpAllocator {
    /// Creates a new empty bump allocator.
    pub const fn new() -> Self {
        BumpAllocator {
            heap_start: 0,
            heap_end: 0,
            next: 0,
            allocations: 0,
        }
    }

    /// Initializes the bump allocator with the given heap bounds.
    ///
    /// This method is unsafe because the caller must ensure that the given
    /// memory range is unused. Also, this method must be called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.heap_start = heap_start;
        self.heap_end = heap_start + heap_size;
        self.next = heap_start;
    }
}
```

`heap_start` 和 `heap_end` 字段跟踪堆内存区域的下边界和上边界。调用者需要确保这些地址有效，否则分配器将返回无效内存。出于这个原因，`init` 函数的调用是 `unsafe` 的。

`next` 字段始终指向堆的第一个未使用字节，即下一次分配的起始地址。因为在开始时整个堆都未被使用，在 `init` 函数中 `next` 设为 `heap_start`。每次分配时，这个字段会按照分配的大小来增长 (_"bumped"_)，以确保不会重复返回相同的内存区域。

`allocations` 字段是一个简单的计数器，记录当前的分配数量，目的是在所有分配都释放后可以重置分配器。它初始化为 0。

我们选择创建一个单独的 init 函数，而不是直接在 new 中执行初始化，是为了保持接口与 linked_list_allocator crate 提供的分配器相同。这样无需额外更改代码即可切换分配器。

### 实现 `GlobalAlloc`

如[前一篇文章][global-alloc]所述，所有堆分配器都需要实现 [`GlobalAlloc`] trait，定义如下：

[global-alloc]: @/edition-2/posts/10-heap-allocation/index.md#the-allocator-interface
[`GlobalAlloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html

```rust
pub unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(
        &self,
        ptr: *mut u8,
        layout: Layout,
        new_size: usize
    ) -> *mut u8 { ... }
}
```

仅 `alloc` 和 `dealloc` 方法是必须的；其他两个方法有默认实现，可以省略。

#### 首次尝试

让我们尝试为 `BumpAllocator` 实现 `alloc` 方法：

```rust
// in src/allocator/bump.rs

use alloc::alloc::{GlobalAlloc, Layout};

unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // TODO alignment and bounds check
        let alloc_start = self.next;
        self.next = alloc_start + layout.size();
        self.allocations += 1;
        alloc_start as *mut u8
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        todo!();
    }
}
```

首先，我们使用 `next` 字段作为分配的起始地址。然后更新 `next` 字段以指向分配的末尾地址，这是堆上的下一个未使用地址。在将分配的起始地址作为 `*mut u8` 指针返回之前，将 `allocations` 计数器加 1。

注意我们没有做任何边界检查或对齐调整，因此这个实现尚不安全。这不要紧，因为现在还无法编译，错误如下：

```
error[E0594]: cannot assign to `self.next` which is behind a `&` reference
  --> src/allocator/bump.rs:29:9
   |
29 |         self.next = alloc_start + layout.size();
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be written
```

(同样的错误也发生在 `self.allocations += 1` 这一行。简洁起见，先省略。)

这个错误是因为 `GlobalAlloc` trait 的 [`alloc`] 和 [`dealloc`] 方法仅作用于不可变的 `&self` 引用，因此无法更新 `next` 和 `allocations` 字段。这是一个问题，因为在每次分配时更新 `next` 是 Bump 分配器的基本原则。

[`alloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.alloc
[`dealloc`]: https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.dealloc

#### `GlobalAlloc` 与可变性

在研究这个可变性问题的解决方案之前，让我们试着理解为什么 `GlobalAlloc` trait 的方法是用 `&self` 定义的：正如在[上一篇文章][global-allocator]中，全局堆分配器的定义方法，是将 `#[global_allocator]` 标记添加到一个实现了 `GlobalAlloc` trait 的 `static` 值。静态变量在 Rust 中是不可变的，因此无法在静态分配器上采用 `&mut self` 的方法。因此，`GlobalAlloc` 的所有方法只能采用不可变的 `&self` 引用。

[global-allocator]:  @/edition-2/posts/10-heap-allocation/index.md#the-global-allocator-attribute

所幸，有一种方法可以从 `&self` 获取到 `&mut self` 的引用：将分配器包装在 [`spin::Mutex`] 自旋锁中，可以使用同步的[内部可变性][interior mutability]。`spin::Mutex` 提供了一个 `lock` 方法，可以实现[互斥][mutual exclusion]，从而安全地将 `&self` 转换为 `&mut self` 引用。我们已经在内核中多次使用了包装类型，例如 [VGA 文本缓冲区][vga-mutex]。

[interior mutability]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[vga-mutex]: @/edition-2/posts/03-vga-text-buffer/index.md#spinlocks
[`spin::Mutex`]: https://docs.rs/spin/0.5.0/spin/struct.Mutex.html
[mutual exclusion]: https://en.wikipedia.org/wiki/Mutual_exclusion

#### `Locked` 包装类型

在 `spin::Mutex` 包装类型的帮助下，我们可以为 Bump 分配器实现 `GlobalAlloc` trait。诀窍是不直接为 `BumpAllocator` 实现 trait。诀窍是不直接为，而是为包装的 `spin::Mutex<BumpAllocator>` 实现 trait：

```rust
unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {…}
```

可惜，这样还不行，因为 Rust 编译器不允许为其他 crate 中定义的类型实现 trait：

```
error[E0117]: only traits defined in the current crate can be implemented for arbitrary types
  --> src/allocator/bump.rs:28:1
   |
28 | unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^--------------------------
   | |                           |
   | |                           `spin::mutex::Mutex` is not defined in the current crate
   | impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

为了解决这个问题，需要在 `spin::Mutex` 基础上创建我们自己的包装类型：

```rust
// in src/allocator.rs

/// A wrapper around spin::Mutex to permit trait implementations.
pub struct Locked<A> {
    inner: spin::Mutex<A>,
}

impl<A> Locked<A> {
    pub const fn new(inner: A) -> Self {
        Locked {
            inner: spin::Mutex::new(inner),
        }
    }

    pub fn lock(&self) -> spin::MutexGuard<A> {
        self.inner.lock()
    }
}
```

这个类型是基于 `spin::Mutex<A>` 的通用包装器。它对被包装类型 `A` 没有任何限制，因此它可以包装所有类型，而不仅仅是分配器。它提供了一个简单的 `new` 构造函数来包装一个给定的值。方便起见，它还提供了一个 `lock` 函数，以调用 `Mutex` 上的 `lock`。由于 `Locked` 类型足够通用，对其他分配器实现也很有用，我们将它放在父模块 `allocator` 中。

#### `Locked<BumpAllocator>` 的实现

`Locked` 类型在我们自己的 crate 中定义（相对于 `spin::Mutex`），因此可以使用它来为 Bump 分配器实现 `GlobalAlloc`。完整实现如下：

```rust
// in src/allocator/bump.rs

use super::{align_up, Locked};
use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr;

unsafe impl GlobalAlloc for Locked<BumpAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let mut bump = self.lock(); // get a mutable reference

        let alloc_start = align_up(bump.next, layout.align());
        let alloc_end = match alloc_start.checked_add(layout.size()) {
            Some(end) => end,
            None => return ptr::null_mut(),
        };

        if alloc_end > bump.heap_end {
            ptr::null_mut() // out of memory
        } else {
            bump.next = alloc_end;
            bump.allocations += 1;
            alloc_start as *mut u8
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        let mut bump = self.lock(); // get a mutable reference

        bump.allocations -= 1;
        if bump.allocations == 0 {
            bump.next = bump.heap_start;
        }
    }
}
```

`alloc` 和 `dealloc` 的第一步都是通过 `inner` 字段调用 [`Mutex::lock`] 方法以获得包装内分配器类型的可变引用。这个实例会保持锁定，直到方法结束，因此在多线程上下文中不会发生数据竞争（我们将很快添加线程支持）。

[`Mutex::lock`]: https://docs.rs/spin/0.5.0/spin/struct.Mutex.html#method.lock

相较于之前的原型，现在的 `alloc` 实现考虑了对齐要求，并且执行了边界检查以确保分配位于堆内存区域内。其中第一步是将 `next` 地址向上舍入到 [`Layout`] 参数指定的对齐方式。`align_up` 函数代码稍后展示。然后用请求的分配大小加上 `alloc_start` 以获得分配的末尾地址。为了防止较大分配时的整数溢出，这里使用 [`checked_add`] 方法。如果发生溢出或者分配的末尾地址大于堆的末尾地址，将返回一个空指针以表示内存不足。否则，更新 `next`，并像以前一样将 `allocations` 计数器加 1。最后，返回转换为 `*mut u8` 指针的 `alloc_start` 地址。

[`checked_add`]: https://doc.rust-lang.org/std/primitive.usize.html#method.checked_add
[`Layout`]: https://doc.rust-lang.org/alloc/alloc/struct.Layout.html

`dealloc` 函数忽略了给定的指针和 `Layout` 参数。它只是减少了 `allocations` 计数器。如果计数器到达 `0`，则意味着所有分配已被释放。这时，它将 `next` 重置为 `heap_start` 地址，以使整个堆内存再次可用。

#### 地址对齐

`align_up` 函数足够通用，可以放入父模块 `allocator` 中。基本实现如下：

```rust
// in src/allocator.rs

/// Align the given address `addr` upwards to alignment `align`.
fn align_up(addr: usize, align: usize) -> usize {
    let remainder = addr % align;
    if remainder == 0 {
        addr // addr already aligned
    } else {
        addr - remainder + align
    }
}
```

该函数首先计算 `addr` 除以 `align` 的[余数][remainder]。如果余数为 `0`，则地址已经与给定对齐方式对齐。否则，通过减去余数（这样新的余数为 0）然后加上对齐（这样地址就不会变得比原来的地址小）来对齐地址。

[remainder]: https://en.wikipedia.org/wiki/Euclidean_division

请注意，这还不是最高效的实现方式。更快的实现如下所示：

```rust
/// Align the given address `addr` upwards to alignment `align`.
///
/// Requires that `align` is a power of two.
fn align_up(addr: usize, align: usize) -> usize {
    (addr + align - 1) & !(align - 1)
}
```

此方法要求 `align` 是 2 的幂，这可以通过 `GlobalAlloc` trait（及其 [`Layout`] 参数）来保证。这使得创建一个[位掩码][bitmask]成为可能，它可以非常高效地对齐地址。让我们从右边部分开始逐步了解这是如何工作的：

[`Layout`]: https://doc.rust-lang.org/alloc/alloc/struct.Layout.html
[bitmask]: https://en.wikipedia.org/wiki/Mask_(computing)

- 由于 `align` 是 2 的幂，因此它的[二进制表示][binary representation]仅有一个位被置为 1（例如 `0b000100000`）。这意味着 `align - 1` 设置了所有低的位（例如 `0b00011111`）。
- 通过 `!` 运算符表示的[位运算 `NOT`][bitwise `NOT`]，可以得到一个数字，除了低于 `align` 的位之外，所有位都被置为了 1（例如 `0b…111111111100000`）。
- 通过对地址和 `!(align - 1)` 执行[位运算 `AND`][bitwise `AND`]，地址被 _向下_ 对齐。这通过清除所有低于 `align` 的位来实现。
- 因为我们想要向上对齐而不是向下对齐，所以在执行按位 `AND` 之前将 `addr` 增加 `align - 1`。这样，已对齐的地址保持不变，而不对齐的地址舍入到下一个对齐边界。

[binary representation]: https://en.wikipedia.org/wiki/Binary_number#Representation
[bitwise `NOT`]: https://en.wikipedia.org/wiki/Bitwise_operation#NOT
[bitwise `AND`]: https://en.wikipedia.org/wiki/Bitwise_operation#AND

你可以自己选择使用哪种方法。两种计算有相同的结果，只是过程不同。

### 使用

要用 Bump 分配器代替 `linked_list_allocator` crate，需要更新 `allocator.rs` 中的 `ALLOCATOR` static 值：

```rust
// in src/allocator.rs

use bump::BumpAllocator;

#[global_allocator]
static ALLOCATOR: Locked<BumpAllocator> = Locked::new(BumpAllocator::new());
```

这里有个重点，我们将 `BumpAllocator::new` 和 `Locked::new` 声明为了 [`const` 函数][`const` functions]。如果它们是普通函数，就会发生编译错误，因为 `static` 的初始化表达式是在编译时求值的。

[`const` functions]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

我们不需要更改 `init_heap` 函数中的 `ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE)` 调用，因为 Bump 分配器有着与 `linked_list_allocator` 相同的接口。

现在内核使用了我们自己的 Bump 分配器！一切仍然可用，包括在上一篇文章中创建的 [`heap_allocation` 测试][`heap_allocation` tests]：

[`heap_allocation` tests]: @/edition-2/posts/10-heap-allocation/index.md#adding-a-test

```
> cargo test --test heap_allocation
[…]
Running 3 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
```

### 讨论

Bump 分配器的一大优势是它非常快。与其他需要主动寻找合适内存块并在 `alloc` 和 `dealloc` 时执行各种记录的分配器设计（见下文）相比，Bump 分配器[可以优化][bump downwards]到仅仅几个汇编指令。这使得 Bump 分配器对于优化分配性能非常有用，例如在创建[虚拟 DOM 库][virtual DOM library]时。

[bump downwards]: https://fitzgeraldnick.com/2019/11/01/always-bump-downwards.html
[virtual DOM library]: https://hacks.mozilla.org/2019/03/fast-bump-allocated-virtual-doms-with-rust-and-wasm/

虽然 Bump 分配器很少用作全局分配器，但 Bump 分配的原理经常以 [Arena 分配][arena allocation]的形式被用到，基本上就是将多次分配合并起来批处理以提高性能。[`toolshed`] crate 中就有一个 Arena 分配器的例子。

[arena allocation]: https://mgravell.github.io/Pipelines.Sockets.Unofficial/docs/arenas.html
[`toolshed`]: https://docs.rs/toolshed/0.8.1/toolshed/index.html

#### Bump 分配器的缺点

Bump 分配器的主要限制是，只有在所有分配都释放后，才能重用已释放的内存。这意味着单个长期分配就会导致内存无法重用。当添加 `many_boxes` 测试的变体时，可以看到这一点：

```rust
// in tests/heap_allocation.rs

#[test_case]
fn many_boxes_long_lived() {
    let long_lived = Box::new(1); // new
    for i in 0..HEAP_SIZE {
        let x = Box::new(i);
        assert_eq!(*x, i);
    }
    assert_eq!(*long_lived, 1); // new
}
```

与 `many_boxes` 测试一样，如果分配器不重用已释放的内存，此测试创建的大量分配就会引发内存不足故障。此外，该测试创建了一个 `long_lived` 分配，它在整个循环执行期间都有效。

当尝试执行新的测试时，我们发现它确实失败了：

```
> cargo test --test heap_allocation
Running 4 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [failed]

Error: panicked at 'allocation error: Layout { size_: 8, align_: 8 }', src/lib.rs:86:5
```

让我们试着详细了解为什么会发生这种失败：首先，在堆的开头创建了 `long_lived` 分配，从而将 `allocations` 计数器加 1。对于循环的每次迭代，都会创建一个短期分配，并且在下一次迭代开始之前释放。这意味着 `allocations` 计数器在迭代开始时暂时增加到 2，并在迭代结束时减少到 1。现在的问题是，Bump 分配器只能在 _所有_ 分配都被释放时，即 allocations 计数器下降到 0 时，才能重用内存。由于这不会在循环结束之前发生，而每次循环迭代时都会分配一个新的内存区域，从而在多次迭代后导致内存不足。

#### 如何修复测试？

有两个潜在的技巧可修复这个 Bump 分配器的测试：

- 可以更改 `dealloc` 以检查释放的分配是否是 `alloc` 返回的最后一个分配，通过将其结束地址与 `next` 指针进行比较。如果相等，可以安全地将 `next` 重置到释放的起始地址。这样，每次循环迭代都会重复使用相同的内存块。
- 可以添加一个 `alloc_back` 方法，该方法使用额外的 `next_back` 字段从堆的 _末尾_ 分配内存。然后可以手动对所有长期分配使用这种分配方法，从而将堆上的短期分配和长期分配分开。请注意，这个方法只有在事先就清楚每个分配的有效期时才有效。这种方法的另一个缺点是手动执行分配很麻烦并且可能不安全。

虽然这两种方法都可以修复测试，但都不是通用解决方案，因为它们只有在非常特殊的情况下才能重用内存。那么问题来了：有没有一个通用的解决方案可以重用 _所有_ 释放的内存？

#### 重用所有释放的内存？

正如在[上一篇文章][heap-intro]中了解的那样，分配可以存在任意长的时间，并且可以以任意顺序释放。这意味着需要跟踪可能无限数量的不连续、未使用的内存区域，就像下面的这个示例：

[heap-intro]: @/edition-2/posts/10-heap-allocation/index.md#dynamic-memory

![](allocation-fragmentation.svg)

该图展示了堆在一段时间内的情况。开始时，整个堆都未被使用，`next` 地址等于 `heap_start`（第 1 行）。然后发生第一次分配（第 2 行）。在第 3 行中，分配了第二个内存块，并释放了第一个内存块。在第 4 行中进行了更多次分配。其中一半的持续时间很短，并且在第 5 行中被释放，然后发生了另一次新分配。

第 5 行展示了根本的问题：有五个不同大小的未使用内存区域，但 `next` 指针只能指向最后一个区域的开头。对于这个例子，虽然可以将其他未使用内存区域的起始地址和大小存放在一个大小为 4 的数组中，但这并不是通用的解决方案，因为可以轻松地创建包含 8、16 或 1000 个未使用内存区域的示例。

通常，如果有数量不确定的项，可以使用一个堆分配的集合类型。但现在的情况是不行的，因为堆分配器不能依赖于自身（会导致无限递归或死锁）。所以，我们需要找一个其他的解决方案。

## 链表分配器

在实现分配器时，对于跟踪任意数量的空闲内存区域，一个常见技巧是将这些区域本身用作后备存储。这利用了这一点：内存区域仍然映射到虚拟地址并由物理帧支持，但不再需要存储其他信息。通过将已释放区域的相关信息存储在这些区域中，可以跟踪无限数量的已释放区域，而无需额外的内存。

最常见的实现方式是在释放的内存中构造一个链表，其中每个节点都是一块释放的内存区域：

![](linked-list-allocation.svg)

列表的每个节点包含两个字段：内存区域的大小和指向下一个未使用内存区域的指针。使用这种方法，只需要一个指向第一个未使用区域的指针（称为 `head`），就可以跟踪所有未使用区域，而不管它们有多少数量。生成的数据结构通常称为[_自由表_][_free list_]。

[_free list_]: https://en.wikipedia.org/wiki/Free_list

正如你可能从名称中猜到的，这就是 `linked_list_allocator` crate 中使用的技术。使用这种技术的分配器通常也称为 _池分配器_。

### 实现

接下来，我们将创建一个简单的 `LinkedListAllocator` 类型，该类型使用上述方法来跟踪已释放内存区域。本文的其他部分不依赖这一部分，因此你可以按需跳过这部分的实现细节。

#### 分配器类型

首先在新的子模块 `allocator::linked_list` 创建一个私有的 `ListNode` struct：

```rust
// in src/allocator.rs

pub mod linked_list;
```

```rust
// in src/allocator/linked_list.rs

struct ListNode {
    size: usize,
    next: Option<&'static mut ListNode>,
}
```

就像上文的图中一样，列表节点有一个 `size` 字段，和一个指向下一个节点的可选指针，由 `Option<&'static mut ListNode>` 类型表示。`&'static mut` 在语义上描述了一个指针下拥有[所有权][owned]的对象。基本上，这相当于一个没有析构函数的 [`Box`]，可以在作用域末尾释放对象。

[owned]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
[`Box`]: https://doc.rust-lang.org/alloc/boxed/index.html

我们为 `ListNode` 实现以下一组方法：

```rust
// in src/allocator/linked_list.rs

impl ListNode {
    const fn new(size: usize) -> Self {
        ListNode { size, next: None }
    }

    fn start_addr(&self) -> usize {
        self as *const Self as usize
    }

    fn end_addr(&self) -> usize {
        self.start_addr() + self.size
    }
}
```

该类型有一个名为 `new` 的简单构造函数，以及计算所表示区域开始和结束地址的方法。这里将 `new` 函数设为 [const 函数][const function]，稍后在构造静态链表分配器时将会用到。请注意，在 const 函数中使用可变引用（包括将 `next` 字段设置为 `None`）仍然不稳定（unstable）。为了通过编译，需要在 `lib.rs` 的开头添加 **`#![feature(const_mut_refs)]`**。

[const function]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

通过将 `ListNode` struct 作为构建的块，现在可以创建 `LinkedListAllocator` struct 了：

```rust
// in src/allocator/linked_list.rs

pub struct LinkedListAllocator {
    head: ListNode,
}

impl LinkedListAllocator {
    /// Creates an empty LinkedListAllocator.
    pub const fn new() -> Self {
        Self {
            head: ListNode::new(0),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.add_free_region(heap_start, heap_size);
    }

    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        todo!();
    }
}
```

该结构包含一个 `head` 节点，指向第一个堆区域。我们只对 `next` 指针的值感兴趣，所以在 `ListNode::new` 函数中将 `size` 设置为 0。这里让 `head` 为 `ListNode` 而不是 `&'static mut ListNode` 类型，以简化 `alloc` 方法的实现。

与 Bump 分配器一样，`new` 函数不使用堆边界初始化分配器。除了保持 API 兼容性之外，还有一个原因是初始化过程中需要将节点数据写入堆内存，而这只能在运行时做到。但是，`new` 函数必须是可以在编译时求值的 [`const` 函数][`const` function]，因为将用于初始化 `ALLOCATOR` static 值。所以，我们再次提供了一个单独的、非 const 的 `init` 方法。

[`const` function]: https://doc.rust-lang.org/reference/items/functions.html#const-functions

`init` 方法中使用了 `add_free_region`，其实现将在稍后展示。现在，我们使用 [`todo!`] 宏，来提供一个总是 panic 的占位符实现。

[`todo!`]: https://doc.rust-lang.org/core/macro.todo.html

#### `add_free_region` 方法

`add_free_region` 方法提供了基本的链表 _push_ 操作。目前我们只从 `init` 调用这个方法，但它也将是 `dealloc` 实现中的核心方法。当分配的内存区域被释放时，会调用 `dealloc` 方法。为了跟踪这块释放的内存区域，可以把它 push 到链表中。

`add_free_region` 方法的实现如下：

```rust
// in src/allocator/linked_list.rs

use super::align_up;
use core::mem;

impl LinkedListAllocator {
    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        // ensure that the freed region is capable of holding ListNode
        assert_eq!(align_up(addr, mem::align_of::<ListNode>()), addr);
        assert!(size >= mem::size_of::<ListNode>());

        // create a new list node and append it at the start of the list
        let mut node = ListNode::new(size);
        node.next = self.head.next.take();
        let node_ptr = addr as *mut ListNode;
        node_ptr.write(node);
        self.head.next = Some(&mut *node_ptr)
    }
}
```

该方法以内存区域的地址和大小作为参数，将其添加到列表中。首先确保给定区域具有存储 `ListNode` 所需的大小和对齐方式，然后创建了一个节点并将其插入到列表中，步骤如下：

![](linked-list-allocator-push.svg)

步骤 0 展示了 `add_free_region` 之前堆的状态。在步骤 1 中，该方法被调用，使用了图中标记为 `freed` 的内存区域。在初始检查之后，该方法在其栈上创建了一个新的 `node`，其大小与已释放区域的大小相同。然后使用了 [`Option::take`] 方法，将节点的 `next` 指针设置为当前的 `head` 指针，同时将 `head` 指针重置为 `None`。

[`Option::take`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.take

在步骤 2 中，该方法通过 [`write`] 方法将新创建的 `node` 写入了被释放内存区域的开头。然后将 `head` 指针指向了新节点。最后生成的指针结构看起来有点混乱，因为释放的区域总是插入到列表的开头，但如果跟随一连串指针的指引，可以看到所有空闲区域都可以从 `head` 指针访问到。

[`write`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write

#### `find_region` 方法

链表的第二个基本操作是查找条目并将其从列表中删除。这是实现 `alloc` 方法所需的核心操作。`find_region` 方法的实现方式如下：

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Looks for a free region with the given size and alignment and removes
    /// it from the list.
    ///
    /// Returns a tuple of the list node and the start address of the allocation.
    fn find_region(&mut self, size: usize, align: usize)
        -> Option<(&'static mut ListNode, usize)>
    {
        // reference to current list node, updated for each iteration
        let mut current = &mut self.head;
        // look for a large enough memory region in linked list
        while let Some(ref mut region) = current.next {
            if let Ok(alloc_start) = Self::alloc_from_region(&region, size, align) {
                // region suitable for allocation -> remove node from list
                let next = region.next.take();
                let ret = Some((current.next.take().unwrap(), alloc_start));
                current.next = next;
                return ret;
            } else {
                // region not suitable -> continue with next region
                current = current.next.as_mut().unwrap();
            }
        }

        // no suitable region found
        None
    }
}
```

该方法使用 `current` 变量和 [`while let` 循环][`while let` loop]来迭代列表元素。一开始，将 `current` 设为 `head` 节点（虚拟节点）。在每次迭代中，它都会被更新为当前节点的 `next`（在 `else` 块中），即下一个节点。如果该区域适合分配，即具有给定大小和对齐方式，则该区域将从列表中删除，并与 `alloc_start` 地址一起返回。

[`while let` loop]: https://doc.rust-lang.org/reference/expressions/loop-expr.html#predicate-pattern-loops

当 `current.next` 指针变为 `None` 时，循环退出。这意味着我们遍历了整个列表，但没有找到适合分配的区域。这种情况下，我们返回 `None`。一个区域是否合适由 `alloc_from_region` 函数来判断，其实现将稍后展示。

让我们更详细地了解一下如何从列表中删除合适的区域：

![](linked-list-allocator-remove-region.svg)

步骤 0 显示了指针调整之前的情况。图中标记了 `region` 和 `current` 区域以及 `region.next` 和 `current.next` 指针。在步骤 1 中，使用了 [`Option::take`] 方法，将 `region.next` 和 `current.next` 指针都置为了 `None`。原始指针存储在了名为 `next` 和 `ret` 的局部变量中。

在步骤 2 中，将 `current.next` 指针设置为局部的 `next` 指针，即原来的 `region.next` 指针。效果是 `current` 现在直接指向 `region` 之后的区域，这样 `region` 就不再是链表的元素了。然后该函数返回存储在局部 `ret` 变量中的 `region` 的指针。

##### `alloc_from_region` 函数

`alloc_from_region` 函数检查了一个区域是否适合给定大小和对齐方式的分配。定义如下：

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Try to use the given region for an allocation with given size and
    /// alignment.
    ///
    /// Returns the allocation start address on success.
    fn alloc_from_region(region: &ListNode, size: usize, align: usize)
        -> Result<usize, ()>
    {
        let alloc_start = align_up(region.start_addr(), align);
        let alloc_end = alloc_start.checked_add(size).ok_or(())?;

        if alloc_end > region.end_addr() {
            // region too small
            return Err(());
        }

        let excess_size = region.end_addr() - alloc_end;
        if excess_size > 0 && excess_size < mem::size_of::<ListNode>() {
            // rest of region too small to hold a ListNode (required because the
            // allocation splits the region in a used and a free part)
            return Err(());
        }

        // region suitable for allocation
        Ok(alloc_start)
    }
}
```

首先，该函数使用之前定义的 `align_up` 函数和 [`checked_add`] 方法计算潜在分配的开始和结束地址。如果发生溢出，或者请求的结束地址在该区域的结束地址之后，则说明请求的分配不适合该区域，此时返回错误。

之后该函数执行了一个看起来作用不明显的检查。但这个检查是必要的，因为大多数时候，请求的分配并不是完全匹配该区域，即分配后还有一部分剩余区域。这部分区域在分配后也需要存储一个 `ListNode`，因此必须有足够的大小。该检查准确地验证了：分配完全匹配（`excess_size == 0`），或者剩余的大小足以存储一个 `ListNode`。

#### 实现 `GlobalAlloc`

通过 `add_free_region` 和 `find_region` 方法提供的基本操作，我们现在终于可以实现 `GlobalAlloc` trait 了。与 Bump 分配器一样，我们不直接为 `LinkedListAllocator` 实现 trait，而只为包装的 `Locked<LinkedListAllocator>` 实现。[`Locked` 包装器][`Locked` wrapper]通过自旋锁添加了内部可变性，即使 `alloc` 和 `dealloc` 方法只接受 `&self` 引用，我们也可以修改分配器实例。

[`Locked` wrapper]: @/edition-2/posts/11-allocator-designs/index.md#a-locked-wrapper-type

其实现如下：

```rust
// in src/allocator/linked_list.rs

use super::Locked;
use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr;

unsafe impl GlobalAlloc for Locked<LinkedListAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // perform layout adjustments
        let (size, align) = LinkedListAllocator::size_align(layout);
        let mut allocator = self.lock();

        if let Some((region, alloc_start)) = allocator.find_region(size, align) {
            let alloc_end = alloc_start.checked_add(size).expect("overflow");
            let excess_size = region.end_addr() - alloc_end;
            if excess_size > 0 {
                allocator.add_free_region(alloc_end, excess_size);
            }
            alloc_start as *mut u8
        } else {
            ptr::null_mut()
        }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // perform layout adjustments
        let (size, _) = LinkedListAllocator::size_align(layout);

        self.lock().add_free_region(ptr as usize, size)
    }
}
```

让我们从 `dealloc` 方法开始，这个比较简单：首先，执行了一些布局调整，稍后将详细解释。然后，通过调用 [`Locked` 包装器][`Locked` wrapper]上的 [`Mutex::lock`] 函数来获取 `&mut LinkedListAllocator` 引用。最后，调用 `add_free_region` 函数将释放的区域添加到空闲列表中。

`alloc` 方法复杂一点。以相同的布局调整开始，也调用了 [`Mutex::lock`] 函数来获取分配器的可变引用。然后使用 `find_region` 方法找到适合分配的内存区域，并将其从列表中删除。如果上一步不成功，拿到了 `None`，这里就会返回 `null_mut`，表示因为没有合适的内存区域而发生了错误。

在成功的情况下，`find_region` 方法返回一个元组，包含合适的区域（已从列表中删除）和分配起始地址。使用 `alloc_start`、分配大小和区域结束地址，再次计算了分配的结束地址和剩余大小。如果剩余的大小不为空，则调用 `add_free_region` 将剩余的内存区域添加回空闲列表。最后，返回转换为 `*mut u8` 指针的 `alloc_start` 地址。

#### Layout Adjustments

So what are these layout adjustments that we make at the beginning of both `alloc` and `dealloc`? They ensure that each allocated block is capable of storing a `ListNode`. This is important because the memory block is going to be deallocated at some point, where we want to write a `ListNode` to it. If the block is smaller than a `ListNode` or does not have the correct alignment, undefined behavior can occur.

The layout adjustments are performed by the `size_align` function, which is defined like this:

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Adjust the given layout so that the resulting allocated memory
    /// region is also capable of storing a `ListNode`.
    ///
    /// Returns the adjusted size and alignment as a (size, align) tuple.
    fn size_align(layout: Layout) -> (usize, usize) {
        let layout = layout
            .align_to(mem::align_of::<ListNode>())
            .expect("adjusting alignment failed")
            .pad_to_align();
        let size = layout.size().max(mem::size_of::<ListNode>());
        (size, layout.align())
    }
}
```

First, the function uses the [`align_to`] method on the passed [`Layout`] to increase the alignment to the alignment of a `ListNode` if necessary. It then uses the [`pad_to_align`] method to round up the size to a multiple of the alignment to ensure that the start address of the next memory block will have the correct alignment for storing a `ListNode` too.
In the second step, it uses the [`max`] method to enforce a minimum allocation size of `mem::size_of::<ListNode>`. This way, the `dealloc` function can safely write a `ListNode` to the freed memory block.

[`align_to`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.align_to
[`pad_to_align`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.pad_to_align
[`max`]: https://doc.rust-lang.org/std/cmp/trait.Ord.html#method.max

### Using it

We can now update the `ALLOCATOR` static in the `allocator` module to use our new `LinkedListAllocator`:

```rust
// in src/allocator.rs

use linked_list::LinkedListAllocator;

#[global_allocator]
static ALLOCATOR: Locked<LinkedListAllocator> =
    Locked::new(LinkedListAllocator::new());
```

Since the `init` function behaves the same for the bump and linked list allocators, we don't need to modify the `init` call in `init_heap`.

When we now run our `heap_allocation` tests again, we see that all tests pass now, including the `many_boxes_long_lived` test that failed with the bump allocator:

```
> cargo test --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

This shows that our linked list allocator is able to reuse freed memory for subsequent allocations.

### Discussion

In contrast to the bump allocator, the linked list allocator is much more suitable as a general-purpose allocator, mainly because it is able to directly reuse freed memory. However, it also has some drawbacks. Some of them are only caused by our basic implementation, but there are also fundamental drawbacks of the allocator design itself.

#### Merging Freed Blocks

The main problem with our implementation is that it only splits the heap into smaller blocks but never merges them back together. Consider this example:

![](linked-list-allocator-fragmentation-on-dealloc.svg)

In the first line, three allocations are created on the heap. Two of them are freed again in line 2 and the third is freed in line 3. Now the complete heap is unused again, but it is still split into four individual blocks. At this point, a large allocation might not be possible anymore because none of the four blocks is large enough. Over time, the process continues, and the heap is split into smaller and smaller blocks. At some point, the heap is so fragmented that even normal sized allocations will fail.

To fix this problem, we need to merge adjacent freed blocks back together. For the above example, this would mean the following:

![](linked-list-allocator-merge-on-dealloc.svg)

Like before, two of the three allocations are freed in line `2`. Instead of keeping the fragmented heap, we now perform an additional step in line `2a` to merge the two rightmost blocks back together. In line `3`, the third allocation is freed (like before), resulting in a completely unused heap represented by three distinct blocks. In an additional merging step in line `3a`, we then merge the three adjacent blocks back together.

The `linked_list_allocator` crate implements this merging strategy in the following way: Instead of inserting freed memory blocks at the beginning of the linked list on `deallocate`, it always keeps the list sorted by start address. This way, merging can be performed directly on the `deallocate` call by examining the addresses and sizes of the two neighboring blocks in the list. Of course, the deallocation operation is slower this way, but it prevents the heap fragmentation we saw above.

#### Performance

As we learned above, the bump allocator is extremely fast and can be optimized to just a few assembly operations. The linked list allocator performs much worse in this category. The problem is that an allocation request might need to traverse the complete linked list until it finds a suitable block.

Since the list length depends on the number of unused memory blocks, the performance can vary extremely for different programs. A program that only creates a couple of allocations will experience relatively fast allocation performance. For a program that fragments the heap with many allocations, however, the allocation performance will be very bad because the linked list will be very long and mostly contain very small blocks.

It's worth noting that this performance issue isn't a problem caused by our basic implementation but a fundamental problem of the linked list approach. Since allocation performance can be very important for kernel-level code, we explore a third allocator design in the following that trades improved performance for reduced memory utilization.

## Fixed-Size Block Allocator

In the following, we present an allocator design that uses fixed-size memory blocks for fulfilling allocation requests. This way, the allocator often returns blocks that are larger than needed for allocations, which results in wasted memory due to [internal fragmentation]. On the other hand, it drastically reduces the time required to find a suitable block (compared to the linked list allocator), resulting in much better allocation performance.

### Introduction

The idea behind a _fixed-size block allocator_ is the following: Instead of allocating exactly as much memory as requested, we define a small number of block sizes and round up each allocation to the next block size. For example, with block sizes of 16, 64, and 512 bytes, an allocation of 4 bytes would return a 16-byte block, an allocation of 48 bytes a 64-byte block, and an allocation of 128 bytes a 512-byte block.

Like the linked list allocator, we keep track of the unused memory by creating a linked list in the unused memory. However, instead of using a single list with different block sizes, we create a separate list for each size class. Each list then only stores blocks of a single size. For example, with block sizes of 16, 64, and 512, there would be three separate linked lists in memory:

![](fixed-size-block-example.svg).

Instead of a single `head` pointer, we have the three head pointers `head_16`, `head_64`, and `head_512` that each point to the first unused block of the corresponding size. All nodes in a single list have the same size. For example, the list started by the `head_16` pointer only contains 16-byte blocks. This means that we no longer need to store the size in each list node since it is already specified by the name of the head pointer.

Since each element in a list has the same size, each list element is equally suitable for an allocation request. This means that we can very efficiently perform an allocation using the following steps:

- Round up the requested allocation size to the next block size. For example, when an allocation of 12 bytes is requested, we would choose the block size of 16 in the above example.
- Retrieve the head pointer for the list, e.g., for block size 16, we need to use `head_16`.
- Remove the first block from the list and return it.

Most notably, we can always return the first element of the list and no longer need to traverse the full list. Thus, allocations are much faster than with the linked list allocator.

#### Block Sizes and Wasted Memory

Depending on the block sizes, we lose a lot of memory by rounding up. For example, when a 512-byte block is returned for a 128-byte allocation, three-quarters of the allocated memory is unused. By defining reasonable block sizes, it is possible to limit the amount of wasted memory to some degree. For example, when using the powers of 2 (4, 8, 16, 32, 64, 128, …) as block sizes, we can limit the memory waste to half of the allocation size in the worst case and a quarter of the allocation size in the average case.

It is also common to optimize block sizes based on common allocation sizes in a program. For example, we could additionally add block size 24 to improve memory usage for programs that often perform allocations of 24 bytes. This way, the amount of wasted memory can often be reduced without losing the performance benefits.

#### Deallocation

Much like allocation, deallocation is also very performant. It involves the following steps:

- Round up the freed allocation size to the next block size. This is required since the compiler only passes the requested allocation size to `dealloc`, not the size of the block that was returned by `alloc`. By using the same size-adjustment function in both `alloc` and `dealloc`, we can make sure that we always free the correct amount of memory.
- Retrieve the head pointer for the list.
- Add the freed block to the front of the list by updating the head pointer.

Most notably, no traversal of the list is required for deallocation either. This means that the time required for a `dealloc` call stays the same regardless of the list length.

#### Fallback Allocator

Given that large allocations (>2&nbsp;KB) are often rare, especially in operating system kernels, it might make sense to fall back to a different allocator for these allocations. For example, we could fall back to a linked list allocator for allocations greater than 2048 bytes in order to reduce memory waste. Since only very few allocations of that size are expected, the linked list would stay small and the (de)allocations would still be reasonably fast.

#### Creating new Blocks

Above, we always assumed that there are always enough blocks of a specific size in the list to fulfill all allocation requests. However, at some point, the linked list for a given block size becomes empty. At this point, there are two ways we can create new unused blocks of a specific size to fulfill an allocation request:

- Allocate a new block from the fallback allocator (if there is one).
- Split a larger block from a different list. This best works if block sizes are powers of two. For example, a 32-byte block can be split into two 16-byte blocks.

For our implementation, we will allocate new blocks from the fallback allocator since the implementation is much simpler.

### Implementation

Now that we know how a fixed-size block allocator works, we can start our implementation. We won't depend on the implementation of the linked list allocator created in the previous section, so you can follow this part even if you skipped the linked list allocator implementation.

#### List Node

We start our implementation by creating a `ListNode` type in a new `allocator::fixed_size_block` module:

```rust
// in src/allocator.rs

pub mod fixed_size_block;
```

```rust
// in src/allocator/fixed_size_block.rs

struct ListNode {
    next: Option<&'static mut ListNode>,
}
```

This type is similar to the `ListNode` type of our [linked list allocator implementation], with the difference that we don't have a `size` field. It isn't needed because every block in a list has the same size with the fixed-size block allocator design.

[linked list allocator implementation]: #the-allocator-type

#### Block Sizes

Next, we define a constant `BLOCK_SIZES` slice with the block sizes used for our implementation:

```rust
// in src/allocator/fixed_size_block.rs

/// The block sizes to use.
///
/// The sizes must each be power of 2 because they are also used as
/// the block alignment (alignments must be always powers of 2).
const BLOCK_SIZES: &[usize] = &[8, 16, 32, 64, 128, 256, 512, 1024, 2048];
```

As block sizes, we use powers of 2, starting from 8 up to 2048. We don't define any block sizes smaller than 8 because each block must be capable of storing a 64-bit pointer to the next block when freed. For allocations greater than 2048 bytes, we will fall back to a linked list allocator.

To simplify the implementation, we define the size of a block as its required alignment in memory. So a 16-byte block is always aligned on a 16-byte boundary and a 512-byte block is aligned on a 512-byte boundary. Since alignments always need to be powers of 2, this rules out any other block sizes. If we need block sizes that are not powers of 2 in the future, we can still adjust our implementation for this (e.g., by defining a second `BLOCK_ALIGNMENTS` array).

#### The Allocator Type

Using the `ListNode` type and the `BLOCK_SIZES` slice, we can now define our allocator type:

```rust
// in src/allocator/fixed_size_block.rs

pub struct FixedSizeBlockAllocator {
    list_heads: [Option<&'static mut ListNode>; BLOCK_SIZES.len()],
    fallback_allocator: linked_list_allocator::Heap,
}
```

The `list_heads` field is an array of `head` pointers, one for each block size. This is implemented by using the `len()` of the `BLOCK_SIZES` slice as the array length. As a fallback allocator for allocations larger than the largest block size, we use the allocator provided by the `linked_list_allocator`. We could also use the `LinkedListAllocator` we implemented ourselves instead, but it has the disadvantage that it does not [merge freed blocks].

[merge freed blocks]: #merging-freed-blocks

For constructing a `FixedSizeBlockAllocator`, we provide the same `new` and `init` functions that we implemented for the other allocator types too:

```rust
// in src/allocator/fixed_size_block.rs

impl FixedSizeBlockAllocator {
    /// Creates an empty FixedSizeBlockAllocator.
    pub const fn new() -> Self {
        const EMPTY: Option<&'static mut ListNode> = None;
        FixedSizeBlockAllocator {
            list_heads: [EMPTY; BLOCK_SIZES.len()],
            fallback_allocator: linked_list_allocator::Heap::empty(),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.fallback_allocator.init(heap_start, heap_size);
    }
}
```

The `new` function just initializes the `list_heads` array with empty nodes and creates an [`empty`] linked list allocator as `fallback_allocator`. The `EMPTY` constant is needed to tell the Rust compiler that we want to initialize the array with a constant value. Initializing the array directly as `[None; BLOCK_SIZES.len()]` does not work, because then the compiler requires `Option<&'static mut ListNode>` to implement the `Copy` trait, which it does not. This is a current limitation of the Rust compiler, which might go away in the future.

[`empty`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.empty

If you haven't done so already for the `LinkedListAllocator` implementation, you also need to add **`#![feature(const_mut_refs)]`** to the top of your `lib.rs`. The reason is that any use of mutable reference types in const functions is still unstable, including the `Option<&'static mut ListNode>` array element type of the `list_heads` field (even if we set it to `None`).

The unsafe `init` function only calls the [`init`] function of the `fallback_allocator` without doing any additional initialization of the `list_heads` array. Instead, we will initialize the lists lazily on `alloc` and `dealloc` calls.

[`init`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.init

For convenience, we also create a private `fallback_alloc` method that allocates using the `fallback_allocator`:

```rust
// in src/allocator/fixed_size_block.rs

use alloc::alloc::Layout;
use core::ptr;

impl FixedSizeBlockAllocator {
    /// Allocates using the fallback allocator.
    fn fallback_alloc(&mut self, layout: Layout) -> *mut u8 {
        match self.fallback_allocator.allocate_first_fit(layout) {
            Ok(ptr) => ptr.as_ptr(),
            Err(_) => ptr::null_mut(),
        }
    }
}
```

The [`Heap`] type of the `linked_list_allocator` crate does not implement [`GlobalAlloc`] (as it's [not possible without locking]). Instead, it provides an [`allocate_first_fit`] method that has a slightly different interface. Instead of returning a `*mut u8` and using a null pointer to signal an error, it returns a `Result<NonNull<u8>, ()>`. The [`NonNull`] type is an abstraction for a raw pointer that is guaranteed to not be a null pointer. By mapping the `Ok` case to the [`NonNull::as_ptr`] method and the `Err` case to a null pointer, we can easily translate this back to a `*mut u8` type.

[`Heap`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html
[not possible without locking]: #globalalloc-and-mutability
[`allocate_first_fit`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.allocate_first_fit
[`NonNull`]: https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html
[`NonNull::as_ptr`]: https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html#method.as_ptr

#### Calculating the List Index

Before we implement the `GlobalAlloc` trait, we define a `list_index` helper function that returns the lowest possible block size for a given [`Layout`]:

```rust
// in src/allocator/fixed_size_block.rs

/// Choose an appropriate block size for the given layout.
///
/// Returns an index into the `BLOCK_SIZES` array.
fn list_index(layout: &Layout) -> Option<usize> {
    let required_block_size = layout.size().max(layout.align());
    BLOCK_SIZES.iter().position(|&s| s >= required_block_size)
}
```

The block must have at least the size and alignment required by the given `Layout`. Since we defined that the block size is also its alignment, this means that the `required_block_size` is the [maximum] of the layout's [`size()`] and [`align()`] attributes. To find the next-larger block in the `BLOCK_SIZES` slice, we first use the [`iter()`] method to get an iterator and then the [`position()`] method to find the index of the first block that is at least as large as the `required_block_size`.

[maximum]: https://doc.rust-lang.org/core/cmp/trait.Ord.html#method.max
[`size()`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.size
[`align()`]: https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.align
[`iter()`]: https://doc.rust-lang.org/std/primitive.slice.html#method.iter
[`position()`]:  https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.position

Note that we don't return the block size itself, but the index into the `BLOCK_SIZES` slice. The reason is that we want to use the returned index as an index into the `list_heads` array.

#### Implementing `GlobalAlloc`

The last step is to implement the `GlobalAlloc` trait:

```rust
// in src/allocator/fixed_size_block.rs

use super::Locked;
use alloc::alloc::GlobalAlloc;

unsafe impl GlobalAlloc for Locked<FixedSizeBlockAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        todo!();
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        todo!();
    }
}
```

Like for the other allocators, we don't implement the `GlobalAlloc` trait directly for our allocator type, but use the [`Locked` wrapper] to add synchronized interior mutability. Since the `alloc` and `dealloc` implementations are relatively large, we introduce them one by one in the following.

##### `alloc`

The implementation of the `alloc` method looks like this:

```rust
// in `impl` block in src/allocator/fixed_size_block.rs

unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
    let mut allocator = self.lock();
    match list_index(&layout) {
        Some(index) => {
            match allocator.list_heads[index].take() {
                Some(node) => {
                    allocator.list_heads[index] = node.next.take();
                    node as *mut ListNode as *mut u8
                }
                None => {
                    // no block exists in list => allocate new block
                    let block_size = BLOCK_SIZES[index];
                    // only works if all block sizes are a power of 2
                    let block_align = block_size;
                    let layout = Layout::from_size_align(block_size, block_align)
                        .unwrap();
                    allocator.fallback_alloc(layout)
                }
            }
        }
        None => allocator.fallback_alloc(layout),
    }
}
```

Let's go through it step by step:

First, we use the `Locked::lock` method to get a mutable reference to the wrapped allocator instance. Next, we call the `list_index` function we just defined to calculate the appropriate block size for the given layout and get the corresponding index into the `list_heads` array. If this index is `None`, no block size fits for the allocation, therefore we use the `fallback_allocator` using the `fallback_alloc` function.

If the list index is `Some`, we try to remove the first node in the corresponding list started by `list_heads[index]` using the [`Option::take`] method. If the list is not empty, we enter the `Some(node)` branch of the `match` statement, where we point the head pointer of the list to the successor of the popped `node` (by using [`take`][`Option::take`] again). Finally, we return the popped `node` pointer as a `*mut u8`.

[`Option::take`]: https://doc.rust-lang.org/core/option/enum.Option.html#method.take

If the list head is `None`, it indicates that the list of blocks is empty. This means that we need to construct a new block as [described above](#creating-new-blocks). For that, we first get the current block size from the `BLOCK_SIZES` slice and use it as both the size and the alignment for the new block. Then we create a new `Layout` from it and call the `fallback_alloc` method to perform the allocation. The reason for adjusting the layout and alignment is that the block will be added to the block list on deallocation.

#### `dealloc`

The implementation of the `dealloc` method looks like this:

```rust
// in src/allocator/fixed_size_block.rs

use core::{mem, ptr::NonNull};

// inside the `unsafe impl GlobalAlloc` block

unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
    let mut allocator = self.lock();
    match list_index(&layout) {
        Some(index) => {
            let new_node = ListNode {
                next: allocator.list_heads[index].take(),
            };
            // verify that block has size and alignment required for storing node
            assert!(mem::size_of::<ListNode>() <= BLOCK_SIZES[index]);
            assert!(mem::align_of::<ListNode>() <= BLOCK_SIZES[index]);
            let new_node_ptr = ptr as *mut ListNode;
            new_node_ptr.write(new_node);
            allocator.list_heads[index] = Some(&mut *new_node_ptr);
        }
        None => {
            let ptr = NonNull::new(ptr).unwrap();
            allocator.fallback_allocator.deallocate(ptr, layout);
        }
    }
}
```

Like in `alloc`, we first use the `lock` method to get a mutable allocator reference and then the `list_index` function to get the block list corresponding to the given `Layout`. If the index is `None`, no fitting block size exists in `BLOCK_SIZES`, which indicates that the allocation was created by the fallback allocator. Therefore, we use its [`deallocate`][`Heap::deallocate`] to free the memory again. The method expects a [`NonNull`] instead of a `*mut u8`, so we need to convert the pointer first. (The `unwrap` call only fails when the pointer is null, which should never happen when the compiler calls `dealloc`.)

[`Heap::deallocate`]: https://docs.rs/linked_list_allocator/0.9.0/linked_list_allocator/struct.Heap.html#method.deallocate

If `list_index` returns a block index, we need to add the freed memory block to the list. For that, we first create a new `ListNode` that points to the current list head (by using [`Option::take`] again). Before we write the new node into the freed memory block, we first assert that the current block size specified by `index` has the required size and alignment for storing a `ListNode`. Then we perform the write by converting the given `*mut u8` pointer to a `*mut ListNode` pointer and then calling the unsafe [`write`][`pointer::write`] method on it. The last step is to set the head pointer of the list, which is currently `None` since we called `take` on it, to our newly written `ListNode`. For that, we convert the raw `new_node_ptr` to a mutable reference.

[`pointer::write`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.write

There are a few things worth noting:

- We don't differentiate between blocks allocated from a block list and blocks allocated from the fallback allocator. This means that new blocks created in `alloc` are added to the block list on `dealloc`, thereby increasing the number of blocks of that size.
- The `alloc` method is the only place where new blocks are created in our implementation. This means that we initially start with empty block lists and only fill these lists lazily when allocations of their block size are performed.
- We don't need `unsafe` blocks in `alloc` and `dealloc`, even though we perform some `unsafe` operations. The reason is that Rust currently treats the complete body of unsafe functions as one large `unsafe` block. Since using explicit `unsafe` blocks has the advantage that it's obvious which operations are unsafe and which are not, there is a [proposed RFC](https://github.com/rust-lang/rfcs/pull/2585) to change this behavior.

### Using it

To use our new `FixedSizeBlockAllocator`, we need to update the `ALLOCATOR` static in the `allocator` module:

```rust
// in src/allocator.rs

use fixed_size_block::FixedSizeBlockAllocator;

#[global_allocator]
static ALLOCATOR: Locked<FixedSizeBlockAllocator> = Locked::new(
    FixedSizeBlockAllocator::new());
```

Since the `init` function behaves the same for all allocators we implemented, we don't need to modify the `init` call in `init_heap`.

When we now run our `heap_allocation` tests again, all tests should still pass:

```
> cargo test --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

Our new allocator seems to work!

### Discussion

While the fixed-size block approach has much better performance than the linked list approach, it wastes up to half of the memory when using powers of 2 as block sizes. Whether this tradeoff is worth it heavily depends on the application type. For an operating system kernel, where performance is critical, the fixed-size block approach seems to be the better choice.

On the implementation side, there are various things that we could improve in our current implementation:

- Instead of only allocating blocks lazily using the fallback allocator, it might be better to pre-fill the lists to improve the performance of initial allocations.
- To simplify the implementation, we only allowed block sizes that are powers of 2 so that we could also use them as the block alignment. By storing (or calculating) the alignment in a different way, we could also allow arbitrary other block sizes. This way, we could add more block sizes, e.g., for common allocation sizes, in order to minimize the wasted memory.
- We currently only create new blocks, but never free them again. This results in fragmentation and might eventually result in allocation failure for large allocations. It might make sense to enforce a maximum list length for each block size. When the maximum length is reached, subsequent deallocations are freed using the fallback allocator instead of being added to the list.
- Instead of falling back to a linked list allocator, we could have a special allocator for allocations greater than 4&nbsp;KiB. The idea is to utilize [paging], which operates on 4&nbsp;KiB pages, to map a continuous block of virtual memory to non-continuous physical frames. This way, fragmentation of unused memory is no longer a problem for large allocations.
- With such a page allocator, it might make sense to add block sizes up to 4&nbsp;KiB and drop the linked list allocator completely. The main advantages of this would be reduced fragmentation and improved performance predictability, i.e., better worst-case performance.

[paging]: @/edition-2/posts/08-paging-introduction/index.md

It's important to note that the implementation improvements outlined above are only suggestions. Allocators used in operating system kernels are typically highly optimized for the specific workload of the kernel, which is only possible through extensive profiling.

### Variations

There are also many variations of the fixed-size block allocator design. Two popular examples are the _slab allocator_ and the _buddy allocator_, which are also used in popular kernels such as Linux. In the following, we give a short introduction to these two designs.

#### Slab Allocator

The idea behind a [slab allocator] is to use block sizes that directly correspond to selected types in the kernel. This way, allocations of those types fit a block size exactly and no memory is wasted. Sometimes, it might be even possible to preinitialize type instances in unused blocks to further improve performance.

[slab allocator]: https://en.wikipedia.org/wiki/Slab_allocation

Slab allocation is often combined with other allocators. For example, it can be used together with a fixed-size block allocator to further split an allocated block in order to reduce memory waste. It is also often used to implement an [object pool pattern] on top of a single large allocation.

[object pool pattern]: https://en.wikipedia.org/wiki/Object_pool_pattern

#### Buddy Allocator

Instead of using a linked list to manage freed blocks, the [buddy allocator] design uses a [binary tree] data structure together with power-of-2 block sizes. When a new block of a certain size is required, it splits a larger sized block into two halves, thereby creating two child nodes in the tree. Whenever a block is freed again, its neighbor block in the tree is analyzed. If the neighbor is also free, the two blocks are joined back together to form a block of twice the size.

The advantage of this merge process is that [external fragmentation] is reduced so that small freed blocks can be reused for a large allocation. It also does not use a fallback allocator, so the performance is more predictable. The biggest drawback is that only power-of-2 block sizes are possible, which might result in a large amount of wasted memory due to [internal fragmentation]. For this reason, buddy allocators are often combined with a slab allocator to further split an allocated block into multiple smaller blocks.

[buddy allocator]: https://en.wikipedia.org/wiki/Buddy_memory_allocation
[binary tree]: https://en.wikipedia.org/wiki/Binary_tree
[external fragmentation]: https://en.wikipedia.org/wiki/Fragmentation_(computing)#External_fragmentation
[internal fragmentation]: https://en.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation


## Summary

This post gave an overview of different allocator designs. We learned how to implement a basic [bump allocator], which hands out memory linearly by increasing a single `next` pointer. While bump allocation is very fast, it can only reuse memory after all allocations have been freed. For this reason, it is rarely used as a global allocator.

[bump allocator]: @/edition-2/posts/11-allocator-designs/index.md#bump-allocator

Next, we created a [linked list allocator] that uses the freed memory blocks itself to create a linked list, the so-called [free list]. This list makes it possible to store an arbitrary number of freed blocks of different sizes. While no memory waste occurs, the approach suffers from poor performance because an allocation request might require a complete traversal of the list. Our implementation also suffers from [external fragmentation] because it does not merge adjacent freed blocks back together.

[linked list allocator]: @/edition-2/posts/11-allocator-designs/index.md#linked-list-allocator
[free list]: https://en.wikipedia.org/wiki/Free_list

To fix the performance problems of the linked list approach, we created a [fixed-size block allocator] that predefines a fixed set of block sizes. For each block size, a separate [free list] exists so that allocations and deallocations only need to insert/pop at the front of the list and are thus very fast. Since each allocation is rounded up to the next larger block size, some memory is wasted due to [internal fragmentation].

[fixed-size block allocator]: @/edition-2/posts/11-allocator-designs/index.md#fixed-size-block-allocator

There are many more allocator designs with different tradeoffs. [Slab allocation] works well to optimize the allocation of common fixed-size structures, but is not applicable in all situations. [Buddy allocation] uses a binary tree to merge freed blocks back together, but wastes a large amount of memory because it only supports power-of-2 block sizes. It's also important to remember that each kernel implementation has a unique workload, so there is no "best" allocator design that fits all cases.

[Slab allocation]: @/edition-2/posts/11-allocator-designs/index.md#slab-allocator
[Buddy allocation]: @/edition-2/posts/11-allocator-designs/index.md#buddy-allocator

## 接下来？

With this post, we conclude our memory management implementation for now. Next, we will start exploring [_multitasking_], starting with cooperative multitasking in the form of [_async/await_]. In subsequent posts, we will then explore [_threads_], [_multiprocessing_], and [_processes_].

[_multitasking_]: https://en.wikipedia.org/wiki/Computer_multitasking
[_threads_]: https://en.wikipedia.org/wiki/Thread_(computing)
[_processes_]: https://en.wikipedia.org/wiki/Process_(computing)
[_multiprocessing_]: https://en.wikipedia.org/wiki/Multiprocessing
[_async/await_]: https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html
