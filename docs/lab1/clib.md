## 测例库介绍

从[这里](https://github.com/LearningOS/2023a-stage3-lab1)可以获取到本次实验需要使用的用户程序。你需要把 `testcases` 目录放到当前实验（也就是 `2023a-rcore-` 开头的这个项目）的根目录下，在 `testcases` 目录下运行 `make build` 即可在 `testcases/build/` 下获得 `42` `hello`两个二进制文件。

#### 测例里都有什么

这是一个极简的测例库，可以编译不依赖于 libc 的 C 程序。事实上，它对标的是比赛初赛测例库（ https://github.com/oscomp/testsuits-for-oskernel/tree/master/riscv-syscalls-testing ）的超级简化版。

我们可以像上一节一样，在 `build` 目录下使用 `riscv64-linux-musl-objdump -ld 42 > debug.S` 检查这个可执行文件的内容。它非常短，只需要有一点 RISC-V 汇编语言基础，都应该能读懂里面发生的每一件事（对照汇编看，这里就不粘一遍代码了）：

- `_start` 是整个程序的入口，它会将 `sp` 复制到 `a0`，相当于函数参数，然后调用 `__start_main`。这个参数并不直接等价于我们熟知的 `argc` 或`argv`，我们下面再详细介绍。

- `__start_main` 是一个库提供的初始化函数。它初始化完成后，会调用 C 代码里用户程序自己的 `main` 函数。

- `main`函数只是返回了一个 `42`，然后 `__start_main` 调用了 `exit`

- `exit` 其实相当于 `sys_exit`，也就是大家在 `ch3` 接触到的第一个 `syscall`，用于退出用户程序。在这段汇编中，它填好参数后会调用 `__syscall1`

- 最后 `__syscall1` 通过 `ecall` 陷入内核，通知内核这个用户程序结束了，且返回值是 `42`。`ecall` 后面的代码不会被执行到。

也可以直接看 C 的源代码。其中比较关键的是 `lib\main.c`：

```c
int __start_main(long *p)
{
    int argc = p[0];
    char **argv = (void *)(p+1);

    exit(main(argc, argv));
    return 0;
}
```

其中输入参数 `p` 是 `_start` 调用 `__start_main` 时给的，实际上就是初始的用户栈指针。

`main.c` 中将初始用户栈指针 `p` 指向的值设为 `argc`，然后将 `p+1` 的值设为 `argv`。也就是说，用户栈上的空间大致是这样的

```
position            content                     size (bytes)
  ------------------------------------------------------------------------
  stack pointer ->  [ argc = number of args ]     4
                    [ argv[0] (pointer) ]         4 (program name)
                    [ argv[1] (pointer) ]         4
                    [ argv[..] (pointer) ]        4 * x
                    [ argv[n - 1] (pointer) ]     4
                    [ argv[n] (pointer) ]         4   (= NULL)

                    [ argv[0] ]                   >=0 (program name)
                    [ '\0' ]                      1
                    [ argv[..] ]                  >=0
                    [ '\0' ]                      1
                    [ argv[n - 1] ]               >=0
                    [ '\0' ]                      1

  ------------------------------------------------------------------------
```

注意**从上到下是地址从低到高，而用户栈向低地址扩展**。栈顶是 `argc`， 接下来是 `argv` 数组，它们依次指向每个真正的 `argv[i]` 字符串。

如果还对 `ch7` 的命令行参数一节有印象，可能很快就会发现，上面的栈排布和 `ch7` 指导书教的不太一样。事实上，上面的排布是符合 ELF 文件规范的（见 [About ELF Auxiliary Vectors](https://articles.manugarg.com/aboutelfauxiliaryvectors.html)），而 `rCore-Tutorial` 指导书里的写法仅供学习参考。

为了支持原生的 Linux 应用，后续最好还是改用这里介绍的写法。

#### 为什么使用这样一个测例项目

为什么我们要绕这么一个大弯，去手写一个测例库呢？你可能会想试试写一个最简单的 `hello world`：

```c
#include <stdio.h>

int main() {
    printf("hello world");
}
```

然后用交叉编译器直接编译后看汇编，其实也没有多少行。但注意，这种方式生成出的可执行程序使用了动态链接，这是一项比赛复赛时才会涉及的功能，目前我们的内核是无法运行这样的程序的。例如在这段汇编中甚至无法找到一个 `ecall`，这不是说这个用户程序不会执行 `syscall`，而只是它跳转到了外部的共享库中执行。如果感兴趣，可以在搜索引擎搜索“动态链接”简单了解。

当然也可以指定静态编译。假设上面的 `hello world` 是 `a.c`，使用 `riscv64-linux-musl-gcc a.c -static` 即可。这样得到的可执行文件不依赖外部库了，但有超过 5000 行，很难调试。你也可以在目前的内核中尝试运行它，看看是否会报错。

总之，使用标准库函数（如 `printf` ）的 C 用户程序不是这次实验要处理的问题。这就是为什么我们手写了一个测例库用于实验。

#### 使用用户态 qemu 进行对拍

测例库里编译的测例都是完全符合规范的 `RISC-V` 可执行程序，所以它当然可以在其他内核上运行。

如果你还记得，在 `rCore-Tutorial`的 `ch0`配环境的时候，安装了 `qemu-riscv64` 和 `qemu-system-riscv64`。后者用于运行实验，而前者实际上是一个用户态模拟器。换而言之，它可以直接运行用户态的 `RISC-V` 程序，我们可以直接把测例文件扔给它。

例如在 `testcases/` 目录下执行 `qemu-riscv64 ./build/hello`，就可以获取正确输出（可以打开 `testcases/src/hello.c` 看看正确输出长什么样）。

同样地，也可以执行 `qemu-riscv64 ./build/42`。这个用户程序在退出时返回了一个 `42`，不过没有打印输出。但我们可以在上面的命令之后立即执行

```bash
echo $?
```

就可以看到返回值 `42`

> `$?` 是一个shell的变量，表示上一条命令的返回值。
> 
> 在这个例子中，具体来说是 `qemu` 的返回值。它执行了我们要求的用户程序，然后把用户程序的返回值作为自己的返回值，推给宿主机。

如此一来，后续我们每次遇到一个新的应用程序，就可以用 `qemu-riscv64` 进行检查，看看正常的“内核”运行它应该是什么样的，然后来推测我们的内核运行同一个测例时出了什么错。

我们把这种调试方式叫做“对拍”。
