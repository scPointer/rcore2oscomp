# 选做实验

**这个实验的详细文档后续再更新**，目前是因为有同学做得太快了，所以暂时先给一个简略的版本：

## 实验准备

任意一个你喜欢的内核，可以是 `rCore-Tutorial` 或者 `Starry` 或者第三章实验中列举的其他内核。

## 实验概述

从内核中分离出一个独立的功能模块，将其作为独立的 `Rust Crate` 发布到 `crates.io`。

> 其实更好的说法应该是“编写一个功能模块”。但在时间有限的情况下，我们推荐大家从已有的内核中去把模块拆出来，只要那个内核的开源协议允许就行。

我们的目标是构建一批通用的 `Rust Crate`，给之后参加操作系统比赛的同学“造轮子”，避免大家在复制粘贴修改别人的代码上花费大量时间。因此以下的附加特性能让你的模块更实用：

- 完善的文档、测试
- 将其移植到另一个 OS，修改尽量少的代码使之正常工作，以此证明你的模块是通用且易用的
- 可单独在用户态测试（即支持 `std/no_std`）

更多信息和要求见下一页

## 参考

发布模块的过程可以参考 [Cargo 手册的这一页](https://rustwiki.org/zh-CN/cargo/reference/publishing.html)

文档的例子可以参考 [`Arceos` 项目](https://github.com/rcore-os/arceos) 的 `crates` 文件夹，比如 [`kernel_guard` 模块](https://github.com/rcore-os/arceos/tree/main/crates/kernel_guard)。

模块移植的例子可以参考 [`rCore-Tutorial-in-single-workspace`](https://github.com/YdrMaster/rCore-Tutorial-in-single-workspace) 和 [一个基于 `rCore-Tutorial ch7` 的修改版](https://github.com/scPointer/rCore-Tutorial-v3/tree/ch7-update)，它们的 `signal` `signal_impl` 模块相同。`rCore-Tutorial-in-single-workspace` 是一个致力于模块化、消除 `rCore-Tutorial` 的多分支的项目，运行相同的测例。你也可以看看这个内核的启动流程和模块划分，跟 `rCore-Tutorial` 还是挺不一样的。 不过这两个内核还是有一些相似性，且 `signal` 模块没有发布，所以这可能不算是一个很好的例子，如果能在比赛内核之间移植模块就更好了。

**如果你实在不知道实现什么**，可以试试[第二章中用到的 `loaders` 模块](https://github.com/scPointer/maturin/tree/master/kernel/src/loaders)，不要担心和别人重复，即使是实现相同的功能，也可以有不一样的接口和文档。

最后，如果你的模块用到了其他项目的代码，别忘了看看它们的开源协议，并据此决定你这个模块的开源协议。