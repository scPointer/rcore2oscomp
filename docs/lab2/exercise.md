# 练习

## 编程作业

可以使用前一个实验完成后的代码来做本实验。如果没有完成前一个实验，也可以从 `rCore-Tutorial ch8` 开始修改，不影响实验本身。如果没有做到 `ch8`，也可以用全新的 `ch7` 来做实验。

在本实验中，你需要：

1. 跟随前面文档的指引修改内核，使得运行测例 `hellostd` 可以输出 `Unsupported syscall_id: 29`。

2. 根据 [通过 Manual Page 添加 Syscall](./syscall.md) 一节中介绍的方法，添加其他 `syscall`，使得测例 `hellostd` 可以正常运行并输出 `hello std`。（提示：只有一个新的 `syscall` 需要实现，其他的都可以用内核中现有 `syscall` 代替或者什么都不做直接返回 `0`）

## 问答作业

1. 查询标志位定义。

标准的 `waitpid` 调用的结构是 `pid_t waitpid(pid_t pid, int *_Nullable wstatus, int options);`。其中的 `options` 参数分别有哪些可能（只要列出不需要解释），用 `int` 的 32 个 bit 如何表示？

你可能需要根据[通过 Manual Page 添加 Syscall](./syscall.md#如何搜索到syscall定义)一小节的指引下载 `musl` 源码，并用[寻找报错位置](./pos.md) 中提到的全局搜索技巧进行分析。

## 报告要求

- 完成编程作业，描述实现思路以及修改的代码。本章没有自动评测，我们会人工检查你的代码实现和报告。
- 完成问答题。
- 推荐使用 `markdown` 格式
- 和 rCore-Tutorial 实验类似，报告放在 `reports/`文件夹下，但命名为 `labr2.md`或`labr2.pdf`
