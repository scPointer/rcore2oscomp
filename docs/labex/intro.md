# 选做实验

**这个实验的详细文档后续再更新**，目前是因为有同学做得太快了，所以暂时先给一个简略的版本：

## 实验准备

本实验会以 `rCore-Tutorial ch8` 为基础，给出一个设计独立内核模块的例子，这需要使用 `rCore-Tutorial` 实验的仓库，但**不依赖于**你在 `rCore-Tutorial` 中完成的任何作业代码。你可以跟着这个例子做，看一看[指导书演示中新增和修改的代码](https://github.com/Godones/rCore-Tutorial-v3/tree/ch8-vfs)，但这不是必要的。

之后，可以任意选择其中一种方式完成实验：

- 需要任意一个内核，可以是 `rCore-Tutorial` 或者 `Starry` 或者第三章实验中列举的其他内核，从其中分离出独立的内核模块。
- 在自己的 `rCore-Tutorial` 上继续开发，添加支持 `FAT32` 文件系统的独立内核模块。

## 实验概述

在[第二章的引入外部代码一节](../lab2/usecode.md)中，以 `loaders` 为例介绍了如何用“复制粘贴代码”的方式引入其它内核的代码实现。这种方式不仅繁琐，而且也缺少统一的规范。`Rust` 提供了[以 `Crate` 为基础的包管理机制](https://kaisery.github.io/trpl-zh-cn/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)，每次编译代码时，编译器 `cargo` 会自动从 [官方包管理平台 crates.io](https://crates.io/) 引用别人写好的库或者可执行程序，在 `rCore-Tutorial` 的 `os/Cargo.toml` 中就可以看到这些依赖库。

本实验将会以虚拟文件系统（`VFS`）和内存文件系统（`RamFs`）为例，介绍如何设计并实现一个独立的内核模块，并提供一些关于编写模块的方法和规范。你可以：

- 以此为基础完成一个支持 `FAT32` 文件系统的独立内核模块：这是比赛初赛的要求的一部分，完成这个模块能节省之后参加初赛时工作量，我们在[第一章实验的介绍](../lab1/intro.md#在实验之后)提到过它。
- 或者从其他内核中分离出一个独立的功能模块，将其作为独立的 `Rust Crate` 发布到 `crates.io`：这相当于帮助同届或者之后的参赛同学“造轮子”，避免大家在复制粘贴修改别人的代码上花费大量时间，他们的内核获奖时也有你的一份功劳。

本章节包含以下内容，供你参考:

1. 简要介绍操作系统与模块划分以及往届内核实现中的模块
2. 介绍拆分模块时的原则和要求
3. 展示如何在 `rCore-Tutorial` 中编写新的独立模块虚拟文件系统（`VFS`）和内存文件系统（`RamFs`），并改造原有的`easy-fs`


## 其他参考资料

发布模块的过程可以参考 [Cargo 手册的这一页](https://rustwiki.org/zh-CN/cargo/reference/publishing.html)

文档的例子可以参考 [`Arceos` 项目](https://github.com/rcore-os/arceos) 的 `crates` 文件夹，比如 [`kernel_guard` 模块](https://github.com/rcore-os/arceos/tree/main/crates/kernel_guard)。

模块移植的例子可以参考 [`rCore-Tutorial-in-single-workspace`](https://github.com/YdrMaster/rCore-Tutorial-in-single-workspace) 和 [一个基于 `rCore-Tutorial ch7` 的修改版](https://github.com/scPointer/rCore-Tutorial-v3/tree/ch7-update)，它们的 `signal` `signal_impl` 模块相同。`rCore-Tutorial-in-single-workspace` 是一个致力于模块化、消除 `rCore-Tutorial` 的多分支的项目，运行相同的测例。你也可以看看这个内核的启动流程和模块划分，跟 `rCore-Tutorial` 还是挺不一样的。 不过这两个内核还是有一些相似性，且 `signal` 模块没有发布，所以这可能不算是一个很好的例子，如果能在比赛内核之间移植模块就更好了。

编写测试的文档可以参考[`Rust Course`的这一节](https://course.rs/test/write-tests.html)。

**如果你实在不知道实现什么**，可以试试拆分[第二章中用到的 `loaders` 模块](https://github.com/scPointer/maturin/tree/master/kernel/src/loaders)或者[编写自己的FAT32 文件系统模块](./vfs.md)，不要担心和别人重复，即使是实现相同的功能，也可以有不一样的接口和文档。

最后，如果你的模块用到了其他项目的代码，别忘了看看它们的开源协议，并据此决定你这个模块的开源协议。