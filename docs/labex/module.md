# 对独立模块的要求

我们的目标是构建一批通用的 `Rust Crate`，给之后参加操作系统比赛的同学“造轮子”，避免大家在复制粘贴修改别人的代码上花费大量时间。因此以下的附加特性能让你的模块更实用：

- 独立发布到包管理平台 `crates.io` ，而不只是基于某个 `git` 仓库
- 覆盖某项内核功能，而不只是底层的一些小数据结构
- 与内核相互独立，不依赖于原本所属内核的结构或者执行逻辑运行
- 完善的文档、测试
- 可单独在用户态测试（即支持 `std/no_std`）

如果有时间，你还可以将模块移植到另一个 OS，修改尽量少的代码使之正常工作，以此证明你的模块是通用且易用的。

## 发布到包管理平台

为了能让更多人用到你的模块，它最好是能独立发布到 `Rust` 的包管理平台 `crates.io` 的：

在 `rCore-Tutorial` 内核的 `os/Cargo.toml` 中，我们使用了很多方便的依赖库：

```toml
[dependencies]
riscv = { git = "https://github.com/rcore-os/riscv", features = ["inline-asm"] }
lazy_static = { version = "1.4.0", features = ["spin_no_std"] }
log = "0.4"
buddy_system_allocator = "0.6"
bitflags = "1.2.1"
xmas-elf = "0.7.0"
virtio-drivers = { git = "https://github.com/rcore-os/virtio-drivers", rev = "4ee80e5" }
easy-fs = { path = "../easy-fs" }
```

其中有几种类型：

- `easy-fs` 是使用相对路径访问的，其实也就相当于是内核项目中的自己的模块，所以不是独立模块
- `riscv` 和 `virtio-drivers` 制定了代码仓库的链接，其中 `virtio-drivers` 还指定了 `commit` 号为 `4ee80e5`，需要依赖这个代码仓库才能拉取模块。如果模块里的接口或者 `feature` 修改了，那依赖这两个模块的代码就可能无法运行（这样的事情真实发生过，这就是为什么使用 `virtio-drivers` 要指定一个老的 `commit`）。所以从某种意义上来说，它们也不是很独立。
- `lazy_static` 指定了版本号和特性，这没有问题。无论它的开发者后续如何修改代码，`1.4.0` 版本的代码已经被固定在了 `crates.io` 上，所以使用这个模块的代码肯定可以正常运行（其实还有个关于 `Rust` 版本号的小问题，后面讲），是独立模块
- 其他模块当然也是是独立模块

### 在发布之前

在实现并打磨好一个模块之后，你需要把它[发布到`crates.io`上](https://rustwiki.org/zh-CN/cargo/reference/publishing.html)。不过在发布前，请确保你的模块的所有依赖都来自 `crates.io` 上，不能依赖像上面所述的 `virtio-drivers` 这样来自代码仓库的模块。另外，你可以在 `cargo.toml` 中的 `keywords` 属性中简要阐述模块的功能，这能使别人更方便地搜索到这个模块。

### 模块的版本号

在 `Cargo.toml` 中标记的模块的版本号使用 `a.b.c` 三位数字，但要小心，版本号的约定默认是“最后一位可更新的”。举个例子，`bitflags = "1.2.1"` 表示可使用 `a=1,b=2,c>=1` 的版本，也即如果 `bitflags` 更新到 `1.2.2`，那么设置 `bitflags = "1.2.1"` 的内核仍然可能拉取这个新版本！如果你想固定只用 `1.2.1`，则需要写成 

```
bitflags = "=1.2.1"
```

或者如果想更精细地控制版本，可以写成类似下面这样的形式

```
bitflags = ">=1.2.2, <1.4.1"
```

## 包含内核功能

上面举 `rCore-Tutorial` 的 `os/Cargo.toml` 例子，其中的模块都提供比较底层的功能，例如 `log` 提供日志输出，`buddy_system_allocator` 提供堆内存分配算法。

我们希望能够产出一些操作系统比赛同学能用得上、覆盖某项内核功能的模块。这个模块不需要特别大，例如“内存管理”“进程管理”，但需要提供比较**通用**的功能：

- 如第二章实验中我们用到的 `loader` 提供 ELF 文件解析，是每支队伍在决赛遇到的第一道坎
- `signal` 信号处理的流程长、文档繁多，但本身的实现不需要很复杂。而且从决赛第一阶段的 `libc-test` 开始，几乎每个应用都会用上信号功能
- `pselect` `ppoll` `epoll` 每个都涉及复杂的 `syscall` 定义，但它们之所以复杂性是因为上层 `syscall` 规范的要求，和下层的内核没有太大关系，所以也可以是比较通用的模块
- 内核中用到的各种锁、开关中断的同步机制。事实上已经有一个[可以参考的例子](https://github.com/DeathWish5/kernel-sync)。

我们认为，如果要拆分一项内核功能作为独立模块，上述这些功能是首选，当然你也可以选择其他功能。

### 底层功能模块

实现负责底层功能的模块**不是**本次实验的内容，但在后续的内核开发中，还是会需要这样的模块作为补充：

1. 负责具体功能、只和内核的一个方面相关的模块：
   1. 如设备驱动，只和外部设备沟通，负责实现内核与设备交流的规范，由设备内部的各种寄存器和内存一起运作。`rCore-Tutorial` 中的 `os/src/drivers/block/virtio_blk.rs` 是一个例子，让内核能够与块设备交流。
   2. 堆和物理页的分配，只和内存管理部分相关。内核中需要这样的模块来划分具体的内存块，如 `rCore-Tutorial` 中的 `os/src/drivers/mm/` 下，`heap.rs` 是堆分配器，负责分配和回收可变大小的内存块。
2. 只实现算法，和功能无关的模块：
   1. 进程的调度分配算法。如 `rCore-Tutorial ch5` 实验中的 `stride` 调度算法，只不过这个实验是直接在内核中修改代码，没有将算法本身独立成模块
   2. `mmap` 操作使用的区间树（`rangemap`）结构。它的定义比较复杂，由 [`mmap` 的文档](https://man7.org/linux/man-pages/man2/mmap.2.html)规定，可以通过[这个模块](https://crates.io/crates/range-action-map)简单了解。
   3. 其他查找、映射操作使用的二叉搜索树。有些 `syscall` 要求内核根据某个 `pid` 找到进程，或者高效维护地址空间上的锁和其他信息。再比如 `rCore-Tutorial ch3` 实验中的 `sysinfo`，假如我们在 `ch4` 之后实现它，就能用数据结构维护它所需要的信息而不是开一个大数组了。

## 与内核相互独立

令模块与内核独立是本次实验中最重要的环节。

我们的目标不是要做一个看起来由很多模块组成的“模块化内核”，而是要做**能在各个内核间通用的模块**。

一个正面例子是 [`Arceos` 项目的 `crates` 目录](https://github.com/rcore-os/arceos/tree/main/crates) 中的一些模块就是通用的，其他内核可以直接使用，而不需要做任何修改。相对的，[`Arceos` 项目的 `modules` 目录](https://github.com/rcore-os/arceos/tree/main/modules) 中的模块就有很多同目录下的依赖关系，它们可以组装构建出一个更灵活的 `Arceos`，但其他内核不太方便直接使用它们，因为这些模块的**组装方式和接口中蕴含了 Arceos 自己的逻辑**。

另一个内核本身做得很好但不太符合我们实验要求的例子是 [`umi` 的模块](https://github.com/js2xxx/umi/tree/master/mizu/lib)。这个内核也把许多功能拆成了模块，但它们相互之间的依赖很多。假如我们想在自己的内核中使用 `umi` 的 [`signal` 这个模块](https://github.com/js2xxx/umi/blob/master/mizu/lib/sygnal/Cargo.toml)，就会在 `Cargo.toml` 中看到它有三个本地依赖：

```
[dependencies]
# Local crates
ksc-core = {path = "../ksc-core"}
ksync = {path = "../ksync"}
rv39-paging = {path = "../paging"}
# External crates
array-macro = "2"
crossbeam-queue = {version = "0", default-features = false, features = ["alloc", "nightly"]}
futures-util = {version = "0", default-features = false, features = ["alloc"]}
log = "0"
spin = "0"
```

它依赖的 `ksc-core` 和 `rv39-paging` 是最底层的模块了，但 `ksync` 又引出了本地依赖。当我们想用 `signal` 时，要么把它下面的一连串依赖都放到自己的内核里，要么需要仔细分析代码，然后用自己内核的功能替换掉 `umi` 的这些功能。这实际上跟我们在 [第二章引入外部代码一节](../lab2/usecode.md) 中做的事情没有区别，所以这些模块不能算是“与内核相互独立”。

本章实验要做的是能在各个内核间通用的模块。你可以从上面给出的这些例子出发，通过修改代码把所有本地依赖拆掉，把它变成一个独立模块，也可以自己另外写一个独立模块。

如果不知道怎么设计通用模块、去除依赖，下面有一些常用的技巧

### `Trait` 和回调函数

如果模块需要依赖一些内核的基本功能，可以使用回调函数和 `Trait` 代替实际依赖。

例如 `rCore-Tutorial` 的 `os/src/drivers/block/virtio_blk.rs` 调用了 `virtio_drivers` 这个外部模块以实现 `VirtIO` 驱动，其中包含：

```rust
use virtio_drivers::{Hal, VirtIOBlk, VirtIOHeader};
......
pub struct VirtioHal;

impl Hal for VirtioHal {
    /// allocate memory for virtio_blk device's io data queue
    fn dma_alloc(pages: usize) -> usize {
        let mut ppn_base = PhysPageNum(0);
        for i in 0..pages {
            let frame = frame_alloc().unwrap();
            if i == 0 {
                ppn_base = frame.ppn;
            }
            assert_eq!(frame.ppn.0, ppn_base.0 + i);
            QUEUE_FRAMES.exclusive_access().push(frame);
        }
        let pa: PhysAddr = ppn_base.into();
        pa.0
    }
    /// free memory for virtio_blk device's io data queue
    fn dma_dealloc(pa: usize, pages: usize) -> i32 {
        let pa = PhysAddr::from(pa);
        let mut ppn_base: PhysPageNum = pa.into();
        for _ in 0..pages {
            frame_dealloc(ppn_base);
            ppn_base.step();
        }
        0
    }
    /// translate physical address to virtual address for virtio_blk device
    fn phys_to_virt(addr: usize) -> usize {
        addr
    }
    /// translate virtual address to physical address for virtio_blk device
    fn virt_to_phys(vaddr: usize) -> usize {
        PageTable::from_token(kernel_token())
            .translate_va(VirtAddr::from(vaddr))
            .unwrap()
            .0
    }
}
```

`Hal` 是外部模块 `virtio_drivers` 中定义的 `Trait`，或者简单理解为“规范”。内核通过定义上述代码中的 `impl Hal for VirtioHal` 来帮驱动分配物理页、转换物理地址与虚拟地址。这些都是内存管理模块中才有的操作， `virtio_drivers` 需要这些操作，但不会直接依赖 `rCore-Tutorial` 的内存管理的代码，而是通过定义 `Trait Hal` 的方式告诉内核“你得帮我实现需要这些功能，我才能正常运作”。

当然，这些 `Trait` 和回调的接口设计需要非常小心。即使是在其他内核中，这些接口也需要很方便就能填上。

### 基于现有的规范

上层的 `syscall` 规范和下层的驱动规范是现成的，例如 [`sys_sigaction` 的定义](https://man7.org/linux/man-pages/man2/rt_sigaction.2.html)中给出了结构 `sigaction{ ... } `，那每个内核中就都应该有这么一个类似的结构，针对这些规范制作相应功能的模块是比较通用的。

反之，如果想针对“线程类”或者“进程类”做模块，那么因为每个内核中它们的结构都不太一样，所以最后设计出的模块肯定很难适用于大部分内核。