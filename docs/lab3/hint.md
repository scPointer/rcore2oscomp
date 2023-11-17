# 实验提示

## 总体提示

- 在内核里修改的文件会被记录到文件系统镜像中，如果你想复原最初的 `testcases/sdcard/` 的内容，可以在 `make run` 前执行 `./build_img.sh sdcard`。
- 你可以使用 `axlog` 提供的 `error!` `warn!` `info!` 等输出，如果希望修改输出的等级，可以看看根目录下 `Makefile` 中的 `LOG` 常量。
- `Starry` 的代码量很大，如果不知道从哪里下手，先试试本章教学的 `strace` 方法，只看可能出问题的 `syscall` 和它们调用的函数。
- 你可以把 `SDCARD_TESTCASES` 常量改为 `busybox sh`，然后亲自进入终端调试。
- `Starry` 的[设计文档](https://azure-stars.github.io/Starry/)可能会对你了解这个内核有一些帮助

实验比指导书中介绍的与 `busybox ls` 相关的调试要简单，看 `strace` 找原因也比较直接，最靠近报错输出的那一两个就是答案，不会要求大家分析大量 `syscall`。

## 流程提示

首先，记得按实验要求修改 `SDCARD_TESTCASES`常量。

1. 使用[定位 `ls` 命令存在的问题](./ls.md)一节中提到的 `syscall` 输出，直接 `make run` 运行，`mv` 会给出如下报错信息

```
mv: can't stat 'bin/abc': No error information
```

看看在这条信息的前面有哪些 `syscall`。然后在你的本机随便找个地方运行 `strace` 测试类似的命令：

```
mkdir bin
touch abc
strace busybox mv abc bin/
```

对比一下，哪个 `syscall` 的返回值可能有问题？你只需要修改一行代码，也可以顺手把附近有相同错误的另一行也改了。

2. 此时 mv 会报错：

```
mv: can't rename 'abc': Operation not permitted
```

这是因为位于 `ulib/axstarry/syscall_fs/src/imp/ctl.rs` 的 `syscall_renameat2` 其实只是个空壳，还没有实现。你需要参考[`syscall` 说明](https://man7.org/linux/man-pages/man2/renameat.2.html) 完成它。

在我们给出的 `ctl.rs` 中已经包含了一些额外的提示信息。

3. 在做到第二个编程任务时，可以再次用 `strace`，尝试理解它的 `syscall` 输出：

Linux 下的 `busybox` 是怎么区分 `bin` 和 `bin/` 的，它怎么知道我们想要把 `def` 放到 `bin/def` 目录下，而不是重命名成一个叫 `bin` 的文件？而内核做错了哪里？

只需要改一个文件的一处代码，需要 5 行左右的修改。