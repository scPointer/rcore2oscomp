## 练习

#### 编程作业

完成 `rCore-Tutorial` 的同学可以从 `ch8` 开始修改。如果没有做到 `ch8`，也可以用全新的 `ch7` 来做实验。

在本实验中，你需要：

1. 跟随前面文档的指引，扩展 `easy-fs-fuse`，使得它可以生成同时包含 Rust 和 C 用户程序的镜像

2. 在 `usershell` 里运行了 `42` 和 `hello` 两个用户程序。`42` 的运行结果是符合预期的，但 `hello` 的结果看起来不太对，你的任务是**修改内核，使得 `hello` 测例给出正常输出**（即给出一行以 `my name is` 开头的输出，且 `exit_code`为0）。

#### 问答作业

1. elf 文件和 bin 文件有什么区别？

Linux 的 file 命令可以检查文件的类型，尝试执行以下命令，描述看到的现象，然后尝试解释输出

```
# 在 user/build/elf/ 下
file ch6_file0.elf
# 在 user/build/bin/ 下
file ch6_file0.bin
riscv64-linux-musl-objdump -ld ch6_file0.bin > debug.S
```

如果对应目录下没有文件，请在 `user/` 目录执行 `make build CHAPTER=8 BASE=2`

#### 报告要求

- 完成编程作业，描述实现思路以及修改的代码。本章没有自动评测，我们会人工检查你的代码实现和报告。
- 完成问答题。
- 推荐使用 `markdown` 格式
- 和 rCore-Tutorial 实验类似，报告放在 `reports/`文件夹下，但命名为 `labr1.md`或`labr1.pdf`
