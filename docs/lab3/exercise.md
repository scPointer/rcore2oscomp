## 练习

#### 编程作业

本实验使用 [`Starry`](https://github.com/Azure-stars/Starry) 内核。你可以 `fork` 这个代码仓库：`https://github.com/Azure-stars/Starry`，然后在 `467c73f` 建立一个新分支：
```bash
git checkout 467c73f
git checkout -b lab3

```

随后就可以开始了。

##### 事先准备


1. 跟随指导书的[使用 strace 调试](./lsdebug.md)一节，修改位于 `ulib/axstarry/syscall_utils/src/ctypes.rs` 的 `struct DirEnt` 的定义和 `set_fixed_part` 函数。
2. 在[这里](https://github.com/LearningOS/2023a-stage3-proj2/tree/lab3)可以获取到本次实验需要使用的代码和用户程序，其中有两个代码文件需要替换掉 `Starry` 中的同名文件。

你还可以跟随指导书修改内核运行的测例、添加调试输出等等，但这不是必须的。在内核里修改的文件会被记录到文件系统镜像中，如果你想复原最初的 `testcases/sdcard/` 的内容，可以在 `make run` 前执行 `./build_img.sh sdcard`。

在 `Starry` 全局搜索 `LAB3` 可以找到要增加的 `syscall` 功能。但除了它以外，**其他文件也有需要修改的地方，但没有 LAB3 标识**。

##### 实验3.1

修改 `ulib/axstarry/syscall_entry/src/test.rs` 的`SDCARD_TESTCASES` 常量为：

```rust
#[allow(dead_code)]
pub const SDCARD_TESTCASES: &[&str] = &[
    "busybox touch abc",
    "busybox mv abc bin/",
    "busybox ls bin/abc",
];
```

其中不能正常工作的命令是 `busybox mv abc bin/`。`busybox touch` 用于创建文件 `abc`，`busybox ls bin/abc` 用于检查文件 `bin/abc` 是否存在。你的任务是修改代码，使得 `busybox ls bin/abc` 正常输出 `bin/abc`。

如果你觉得调试过于困难，可以看看下一页的[提示](./hint.md)。

##### 实验3.2

修改 `ulib/axstarry/syscall_entry/src/test.rs` 的`SDCARD_TESTCASES` 常量为：

```rust
#[allow(dead_code)]
pub const SDCARD_TESTCASES: &[&str] = &[
    "busybox touch def",
    "busybox mv def bin",
    "busybox ls bin/def",
];
```

其中不能正常工作的命令是 `busybox mv def bin`。它和实验3.1的区别在于，`bin` 目录后没有斜线 `/`。你的任务是修改代码，使得 `busybox mv def bin` 正常输出 `bin/def`。

如果你觉得调试过于困难，可以看看下一页的[提示](./hint.md)。

#### 问答作业

本章的问答作业是藏在指导书中的几个思考题：

- **思考题1.1** **思考题1.2** 在[“探索 `cargo` 的缓存”一节](./cache.md)的末尾
- **思考题2** 在[“如何使用往届内核”一节](./justrun.md)的[寻找运行测例的位置](./justrun.md#寻找运行测例的位置)段落中
- **思考题3.1** **思考题3.2** 在[“如何使用往届内核”一节](./justrun.md)的末尾

#### 报告要求

- 完成编程作业，描述实现思路以及修改的代码。本章没有自动评测，我们会人工检查你的代码实现和报告。
- 完成问答题。
- 推荐使用 `markdown` 格式
- 和 rCore-Tutorial 实验类似，报告放在 `reports/`文件夹下，但命名为 `labr3.md`或`labr3.pdf`
