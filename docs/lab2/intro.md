# 运行带标准库的 C 程序

建议至少做完本指导书的 `lab1` 再做这个实验。

实验需要使用你在 `rCore-Tutorial` 实验中的仓库，但**不依赖于**你在 `rCore-Tutorial` 中完成的任何作业代码。你可以用自己的 `rCore-Tutorial ch8` 分支来做这个实验中的代码修改，也可以用没有修改过的 `ch7` 分支。

## 实验准备

从[这里](https://github.com/LearningOS/2023a-stage3-proj2/tree/lab2)获取本次实验需要使用的用户程序。你需要把 `lab2` 分支下的 `testcases` 目录放到当前实验（也就是 `2023a-rcore-` 开头的这个项目）的根目录下，在 `testcases` 目录下运行 `make build` 即可在 `testcases/build/` 下获得 `42` `hello` `hellostd` 三个二进制文件。

本次实验中我们只使用 `hellostd` 这个测例。

## 实验概述

在上一个实验的[为什么使用这样一个测例项目](../lab1/clib.md#为什么使用这样一个测例项目) 这一小节中，我们尝试直接交叉编译一个 `hello world`：

```c
#include <stdio.h>

int main() {
    printf("hello world");
}
```

发现内核无法运行这个程序，并且反汇编的结果太长，似乎完全无法调试。

在这次实验中，我们会一步步尝试分析这个测例的报错情况，继续修改内核，使得内核可以成功运行一个带标准库的 `hello world`。

这一章的文档可能在某种程度上更像一个“debug 记录”。指导书里会把难处理的地方都讲一遍，最后留一个比较简单的小实验作为作业。你**可以直接跳到最后一节去完成实验内容，但我们更希望大家可以跟着指导书走一遍调试过程**，学习途中的调试思路。

## 在实验之后

你可以以本实验为基础，完成比赛决赛第一阶段要求的 `libc-test`，测例见[这里](https://github.com/oscomp/testsuits-for-oskernel/tree/libc-test/libc-test)。这个测例集分两个部分，分别是静态链接的测例（`gcc` 编译加 `-static`选项）和动态链接的测例。

##### 支持 libc-test 中的静态链接测例

支持静态链接测例的思路和实验类似。但所有测例本身都会用到 `runtest.exe`，它的[源代码](https://github.com/oscomp/testsuits-for-oskernel/blob/libc-test/libc-test/src/common/runtest.c)中包含了许多复杂的 syscall。

你可以使用本实验中的分析方法想办法跳过它们，也可以尝试使用[这里](https://github.com/scPointer/maturin/blob/master/tools/libc/README.md)提到的思路，逐个击破一些小测例，最后再使用 `runtest.exe` 运行。

##### 支持 libc-test 中的动态链接测例

支持动态链接测例时，需要内核在ELF加载器中主动添入GOT表和PLT表项。如果觉得太难，也可以直接参考从往届代码的实现，比如本实验引用的[这个内核](https://github.com/scPointer/maturin/) 就在 `kernel/src/loaders/mod.rs` 模块中带了动态链接的实现。
