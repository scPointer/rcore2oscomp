# strace 简介

我们之所以大费周章把 syscall 调用的信息都列出来，就是为了向大家介绍使用 `strace` 工具进行调试的方法。`strace` 是 Linux 下的一个调试分析工具，可以追踪程序运行时调用的 `syscall` 和可接收的信号。

通常来说，如果一个测例无法通过，我们可以直接看测例的源代码，找找哪一行出的问题。但如果源代码太长或者根本不可读，`strace` 就非常有用。如果发现本机还没有安装 `strace`，可以通过类似指令安装

```bash
sudo apt install strace
```

上述是 `apt` 包管理器时的安装方式，如果你本机的操作系统使用其他包管理器，可以替换成类似的命令。

的使用方式很简单，直接把 `strace` 加在要运行的命令之前，它就会自动输出所有运行过程中的 `syscall`。例如在本机上运行

```bash
strace touch a
```

就能看到创建文件 `a` 过程中 `touch` 这个用户程序执行的所有 `syscall`。

`strace` 也可以加一些参数，常用的有

- `-c` 按种类统计 `syscall` 的执行时间、次数和报错次数。注意，“报错”只是代表返回小于 0 的错误码，不代表用户程序出错。例如检查文件是否不存在时也可以用 `sys_openat` ，得到 `ENOENT`（没有此文件）的结果属于“报错”，但这就是我们预期的结果。
- `-p <PID>` 表示指定追踪的进程ID。如果想调试一个大的应用程序，可能会有许多进程共同协作，我们可以指定关心某一个
- `-t` 输出时间信息；`-T` 显示每次调用的时间
- `-f` 追踪 `fork`；`-F` 追踪 `fork` 和 `vfork`
- `-e [!]value1[,value2]` 指定要追踪的 `syscall`，如 `-e clone,read` 就是只看 `sys_clone` 和 `sys_read`。反之，前面加感叹号 `!` 表示不追踪对应的 `syscall`
  - 一些终端的实现会将 `!` 理解为其他含义，因此在表达式中使用时需要加反斜杠 `\` 转义。例如不看 `sys_read` 就是 `-e \!read`

更多参数可以通过 `strace -h` 或者 `man strace` 查看。

在比赛中，用 `strace` 调试的好处是可以得到一个来自 Linux 的“**标准答案**”，而往届内核只能算是“优秀学生作业”。但 `strace` 只给出答案不给出解题过程，所以你还是有必要阅读往届内核的代码。由于 `riscv64` 架构和本机不同，我们可以有很多种使用 strace 的不同方式

##### 在 `linux-riscv64` 中运行

比赛测例仓库提供了一种[编译 riscv64 架构的 linux 的方法](https://github.com/oscomp/testsuits-for-oskernel/tree/main/riscv-linux-rootfs)。你可以按照比赛文档的描述，编译一个 linux，然后把它放到 `Qemu` 中运行，就像在 `Qemu` 中运行 `rCore-Tutorial` 和 `Starry` 一样。然后在这个 `linux-riscv64` 中联网下载 `strace`，并把测例都传到里面，就可以在里面用 `strace` 调试所有测例了。

这样做的好处是调试本身很方便，但是编译 Linux、配置 `qemu-riscv64` 中的环境、传数据的过程相对比较麻烦。而且 `Qemu` 里运行一个 Linux 的效率会比较低。

##### 在 `qemu-riscv64` 中运行静态链接测例

[第一章的测例库介绍一节](../lab1/clib.md)的末尾介绍过用户态的 `qemu`。我们可以直接用 `strace` 来调试它，例如对应第二章的测例可以用 `strace qemu-riscv64 hellostd`。

为了不给还没做完 `lab2` 先来看 `lab3` 实验的同学剧透，这里就不放具体输出了。

这样做的好处是不用重新配环境去编译 Linux 了，但是输出本质上是 `qemu-riscv64` 这个程序对我们本机调用的 `syscall` ，而不是 `hellostd` 对 `qemu-riscv64` 模拟的内核调用的 `syscall`。好在 `qemu-riscv64` 会直接把绝大部分的 `syscall` 直接翻译转到本机，所以抛开前面一长串 `qemu-riscv64` 的初始化，你还是可以在输出的最后看到第二章实验中熟悉的 `syscall`。

##### 在 `qemu-riscv64` 中运行动态链接测例

如果想用这个方式运行动态链接的测例，会遇到一些小麻烦。具体来说，比赛的测例仓库的[`libc-test` 说明](https://github.com/oscomp/testsuits-for-oskernel/tree/final-2023/libc-test)会告诉你动态编译的测例需要依赖一个叫 `libc.so` 的动态库文件。所以用 `qemu-riscv64` 运行动态链接测例时，`qemu-riscv64` 会发现缺失这个动态库，然后去**本机的** `/bin/libc.so` 找这个文件。

一般来说，本机是 `x86_64` 或者 `arm64` 架构，不会在 `/bin` 下放一个 `riscv64` 的可执行文件。但如果按照[`libc-test` 说明](https://github.com/oscomp/testsuits-for-oskernel/tree/final-2023/libc-test)里把提到的 `/lib/ld-musl-riscv64-sf.so.1` 放到 `/bin` 下命名为 `libc.so`，确实是可以成功运行测例的。

当然，如果不想这样破坏本机的环境也有办法。可以使用 [maturin 提供的这个模块](https://github.com/scPointer/maturin/tree/master/tools/libc) 编译动态的 `libc-test` 测例，然后把 `libc.so` 和其他动态库复制到你运行 `qemu-riscv64` 的同目录下即可。例如你想在本机的 `./build` 下运行动态链接的测例，就把这个模块里提供的所有 `.so` 复制一份到 `./build` 目录下，然后 `cd build` 再执行 `strace qemu-riscv64 ...` 即可。

> 这个方法之所以有效，是因为在模块的 `Makefile` 下编译时指定了动态库的地址
> 
> ```makefile
> #动态测例
> %.dout : %.c
>     $(CC) $(LDFLAGS) $(CFLAGS) $^ -DDYNAMIC $(COMMON_SRCS) -Lsrc/functional -Lsrc/regression -ltls_align_dso -ltls_init_dso -ldlopen_dso -ltls_get_new-dtv_dso -Wl,-rpath ./ -o $@
>     mkdir -p build
>     cp $@ $(addprefix build/,  $(subst -,_,$(notdir $@)))
> ```
> 
> 其中 `-Wl,-rpath ./` 就是告诉编译器，编译出的动态链接测例去 `./` 目录找动态库，而不是去根目录或者 `/bin`

##### 在本机运行

涉及具体文件操作时，上面的方法就不一定管用了，因为 `qemu-riscv64` 只是运行一个测例文件，没有加载文件系统镜像。

例如我们目前遇到的 `busybox ls` 的问题，只好在本机 `x86_64` 或者 `arm64` 上试一试用 `strace` 调试相同的命令看结果如何了。当然此时 `busybox ls` 的行为不一定和 `riscv64` 的一致，但还是可以利用它的输出来辅助我们的判断。

下一节，我们会实际用这种方式来调试当前的 bug。