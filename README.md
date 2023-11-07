# 从 rCore-Tutorial 到实际应用

在[这里](https://scpointer.github.io/rcore2oscomp/)可以看到本项目的在线文档。

本教程是 rCore-Tutorial 的后续教程，希望能帮助大家从 rCore-Tutorial 以及其他 Rust 内核出发，构建一个可运行原生 Linux 应用、拥有更强大功能的内核。如果你是希望参加[全国大学生计算机系统能力大赛操作系统设计赛](https://os.educg.net/#/)的同学，可以通过本教程学习如何构建自己的内核；如果你是其他 Rust 开发者，也可以通过本教程扩展 [rCore-Tutorial](https://rcore-os.cn/rCore-Tutorial-Book-v3/)，用学到的知识为开源社区做贡献。

本教程的各个实验间**不是连续的**。我们会选择内核开发中的一些小切片，以它们为例设计代码量适中的实验，引导大家在OS比赛中的开发。如果说 `rCore-Tutorial` 是十层居民楼的毛坯房，需要大家通过 5 个实验去“装修”，那么本教程就像是一个个更高楼房的地基，指引大家从这里开始建造摩天大楼。

目前本教程被用作开源操作系统训练营（见 [github 首页说明](https://github.com/LearningOS)）的实践项目。

## 为什么会有这个教程？

[rCore-Tutorial](https://rcore-os.cn/rCore-Tutorial-Book-v3/) 是一个用 Rust 写操作系统的教程。当你做完这个教程，就已经获得了一个支持文件系统、线程等特性，包含约 50 个 `syscall` 的操作系统内核。

不过，如果想要进一步扩展这个内核，就会出现其他问题：

- rCore-Tutorial 的文件系统、测例自成一体，最后完成的内核也只能运行它自己的文件系统、跑自己的测例，和通用测例集乃至真正的 Linux 应用不兼容。
- `syscall` 并不完全和 `POSIX` 接口兼容。部分 `syscall` 是来自 `Zircon`，还有一部分是为了实验考察而添加的仅用于实验本身的 `syscall`
- 模块划分不完善，模块化程度不高，沿用框架开发到后期会比较痛苦

有经验的同学可以自行重构 rCore-Tutorial，或者重新写一个内核来参加OS比赛。但大多数只学过 rCore-Tutorial 的同学来说，这样的重构并不在它教程的范围内，因此修改  rCore-Tutorial  就成了参赛的一道门槛。

本教程的目的是帮助大家跨过这道门槛，能够将更多精力投入到内核本身的开发中。

## 实验形式

本教程中的实验参考 rCore-Tutorial，分为代码实验和思考题，每一章节需要提交报告。但也有以下不同：

- 整个教程没有统一的仓库，部分实验需要使用 rCore-Tutorial 仓库，部分实验自带另一个仓库。
- 实验参考了 [rustlings](https://github.com/rust-lang/rustlings) 教程的形式，边做边学，跟随教程指引就可以完成大部分代码工作，而章节末尾的代码实验预期会需要更少的时间。
