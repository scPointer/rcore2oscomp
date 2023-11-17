# 定位 `ls` 命令存在的问题

做了这么多准备，我们终于可以引出本章节的实验了。目前在 `467c73f` 这个版本的 `Starry` 中，使用 `busybox ls` 和 `busybox mv` 操作时会出现一些问题。类似前两章实验的形式，我们会带着大家修正 `busybox ls` 的问题，然后将 `busybox mv` 留作实验任务。

> 本章节涉及的 `ls` 和 `mv` 是两个 Linux 命令
> 
> `ls` 可以列出当前目录下的文件。例如你可以在本机的终端尝试 `ls` 然后按回车。
> `mv` 表示移动文件，类似于“剪切粘贴+重命名”。例如 `mv ./a/b ../c/d` 表示将当前目录的 `a` 子目录下的 `b` 文件剪切，然后粘贴到到上一级目录的 `c` 子目录下，命名为 `d` 文件。

## busybox 简介

`busybox` 是一个工具包，既包含 Shell 命令语言的实现，也包含了一些常用的 Linux 命令。你可以尝试执行 `busybox` 然后回车，检查它的版本。如果发现本机还没有安装 `busybox`，可以通过类似指令安装

```bash
sudo apt install busybox-static
```

上述是 `apt` 包管理器时的安装方式，如果你本机的操作系统使用其他包管理器，可以替换成类似的命令。

`static` 表示静态链接的版本。如果你直接安装 `sudo apt install busybox` 也是可以的，那样会得到一个动态链接的版本。在日常使用上，这两个版本没有大的区别。但后续的实验会使用 `strace` 来调试它的输出，静态的版本的输出会简洁一些，所以我们**推荐安装静态版本**。

### 工具包

当我们在执行像 `ls` `mv` 这样的指令的时候，实际在执行什么？可以尝试在本机上执行下面的命令

```bash
where ls
```

一般来说输出里会有一项 `/bin/ls`。我们用 `file /bin/ls` 检查这个文件的属性，会发现它真的是一个可执行文件。而 `busybox` 相当于把这些文件都包在一起，运行 `busybox ls` 的输出相当于运行 `ls` 的输出，运行 `busybox mv` 的输出相当于运行 `mv` 的输出。

其中的区别在于，原本的 `ls` 是一个独立的文件，但 `busybox ls`，表示运行 `busybox` 这个程序，`ls` 只是它的参数。即 `argv[0]="busybox",argv[1]="ls"`。

### Shell 实现

开一个新终端，然后敲 `busybox sh` 按回车，你就可以得到一个由 `busybox` 提供的终端。这个终端可能比本来 Linux 自带的 `bash` （`Ubuntu` 下则是 `dash`）简陋一些，但也已经比 `rCore-Tutorial` 里用 Rust 写的那个 `usershell` 强大太多了。你在 `bash` 下能做的大部分操作也可以在 `busybox sh` 下完成。

现在可以把这个 `busybox` 终端关掉了，因为我们将在 `Starry` 中使用 `busybox`。

## 定位问题

首先我们修改 `Starry` 启动时运行的“测例”为 `busybox sh`。因为文件系统镜像里包含 `testcases/sdcard/`，在这个文件夹下确实有 busybox，所以这么运行是没问题的。也即修改 `ulib/axstarry/syscall_entry/src/test.rs` 的`SDCARD_TESTCASES` 为

> 退一万步讲，就算镜像里没有 `busybox`，你也可以在网上下载 `busybox` 源码然后用 `riscv64-linux-musl-gcc` 交叉编译一个出来，放到文件系统镜像对应的目录里。

```rust
pub const SDCARD_TESTCASES: &[&str] = &[
    "busybox sh",
];
```

然后记得**把上一节中的 `syscall` 调试输出和 `use axlog::error;` 注释掉**。因为在终端中运行命令会产生大量的 `syscall`，终端会被输出塞满。

然后 `make run` 后尝试依次运行（目录和文件名是随意取的，你可以换一个）：

```bash
busybox ls
busybox touch 123
busybox ls
busybox mkdir aaa
busybox ls
```

可以观察到 `ls` 每次的输出都没有变化。看上去好像 `touch` 创建文件和 `mkdir` 创建文件夹都失败了。但再执行几条命令呢

```bash
echo "xxx">>123
busybox ls
busybox cat 123
cd aaa
busybox pwd
cd ..
busybox ls
```

这段命令尝试用 `echo` 向文件 `123` 中写入几个字符 `xxx`，然后用 `cat` 命令读取文件内容，是可以正确读取的。又尝试进入 `aaa` 目录并用 `pwd` 打印当前目录，发现 `aaa` 目录也是存在的。看来之前的 `touch` `mkdir` 应该没问题，有问题的是 `ls`。

> 大部分命令都有前缀 `busybox` 但不是全部。这是因为 `Starry` 默认把一部分命令软链接到了 `busybox`，而且在执行文件不存在时尝试把它交给 `/busybox` （也是 `/sh`）解析。这些操作是在 `ulib/axstarry_syscall_entry/src/test.rs` 的 `fs_init` 函数中完成的。

再仔细对比 `busybox ls` 的输出和 `testcases/sdcard/` 里的内容，会发现其中少了几项。 `testcases/sdcard/`  中按字典序排列的最后几个文件在 `busybox ls` 的输出里是没有的，但我们还是可以用类似上面的命令确认这些文件确实存在。 如果不在根目录下，而是换个文件少的目录，`ls` 的行为是正常的，可以看到所有创建的文件和目录。

所以 `busybox ls` 的问题在于它只输出了目录下的部分内容，没有输出全部文件。

### 只获取 `busybox ls` 的输出

接下来我们希望单独分析 `busybox ls` 的 `syscall` 输出结果。为了避免来自终端 `busybox sh` 的 `syscall` 干扰，我们再一次修改 `axstarry_syscall_entry/src/test.rs` 的`SDCARD_TESTCASES` 为

```rust
pub const SDCARD_TESTCASES: &[&str] = &[
    "busybox ls",
];
```

然后恢复之前注释掉的 `use axlog::error;` 和 `error!` 输出，把 `error!` 换个位置，写成：

```rust
/// axstarry_syscall_entry/src/syscall.rs
use syscall_utils::deal_result;

use axlog::error;

#[no_mangle]
pub fn syscall(syscall_id: usize, args: [usize; 6]) -> isize {
    #[cfg(feature = "futex")]
    syscall_task::check_dead_wait();
    let ans = loop {
        #[cfg(feature = "syscall_net")]
        {
            if let Ok(net_syscall_id) = syscall_net::NetSyscallId::try_from(syscall_id) {
                error!("[syscall] id = {:#?}, args = {:?}, entry", net_syscall_id, args);
                break syscall_net::net_syscall(net_syscall_id, args);
            }
        }

        #[cfg(feature = "syscall_mem")]
        {
            if let Ok(mem_syscall_id) = syscall_mem::MemSyscallId::try_from(syscall_id) {
                error!("[syscall] id = {:#?}, args = {:?}, entry", mem_syscall_id, args);
                break syscall_mem::mem_syscall(mem_syscall_id, args);
            }
        }

        #[cfg(feature = "syscall_fs")]
        {
            if let Ok(fs_syscall_id) = syscall_fs::FsSyscallId::try_from(syscall_id) {
                error!("[syscall] id = {:#?}, args = {:?}, entry", fs_syscall_id, args);
                break syscall_fs::fs_syscall(fs_syscall_id, args);
            }
        }

        // #[cfg(feature = "syscall_task")]

        {
            if let Ok(task_syscall_id) = syscall_task::TaskSyscallId::try_from(syscall_id) {
                error!("[syscall] id = {:#?}, args = {:?}, entry", task_syscall_id, args);
                break syscall_task::task_syscall(task_syscall_id, args);
            }
        }

        panic!("unknown syscall id: {}", syscall_id);
    };

    let ans = deal_result(ans);
    error!("[syscall] id = {}, args = {:?}, return {}", syscall_id, args, ans);
    ans
}
```

可以注意到函数开头的 `error!` 移动到了 `match` 的每一项中，并且 `syscall_id` 的输出从 `{}` 换成了 `{:#?}`，这表示输出这个枚举体（即 `enum`）对应的名字。例如 `NetSyscallId` 定义为

```rust
pub enum NetSyscallId {
    // Socket
    SOCKET = 198,
    BIND = 200,
......
```

如果 `net_syscall_id` 的值是 `NetSyscallId::SOCKET`，那么 `println!("{}", net_syscall_id)` 输出 `198`，而 `println!("{:#?}", net_syscall_id)` 输出 `SOCKET`。

这对实际的代码没有影响，只是为了输出 `syscall` 的名字，让我们看输出更方便。

现在 `make run` 后可以得到一长串的 `syscall` 输出，为了简便我们去掉了前缀的输出信息（就是像 `58.641175 0:6 syscall_entry::syscall:29` 这样的前缀），且只截取其中有意义的部分：

```
...... 省略进程初始化

[syscall] id = FSTATAT, args = [18446744073709551516, 1230088, 1073740272, 0, 1230088, 18446744073709551516], entry
[syscall] id = 79, args = [18446744073709551516, 1230088, 1073740272, 0, 1230088, 18446744073709551516], return 0
[syscall] id = OPENAT, args = [18446744073709551516, 1230088, 622592, 0, 0, 0], entry
[syscall] id = 56, args = [18446744073709551516, 1230088, 622592, 0, 0, 0], return 3
[syscall] id = FCNTL64, args = [3, 2, 1, 0, 0, 3], entry
[syscall] id = 25, args = [3, 2, 1, 0, 0, 3], return -22
[syscall] id = GETDENTS64, args = [3, 4344, 2048, 0, 0, 4320], entry
[syscall] id = 61, args = [3, 4344, 2048, 0, 0, 4320], return 2022
[syscall] id = FSTATAT, args = [18446744073709551516, 6432, 1073740144, 256, 6432, 18446744073709551516], entry
[syscall] id = 79, args = [18446744073709551516, 6432, 1073740144, 256, 6432, 18446744073709551516], return 0

...... 省略若干条 FSTATAT

[syscall] id = MMAP, args = [0, 4096, 3, 34, 18446744073709551615, 0], entry
[syscall] id = 222, args = [0, 4096, 3, 34, 18446744073709551615, 0], return 8192
[syscall] id = FSTATAT, args = [18446744073709551516, 8032, 1073740144, 256, 8032, 18446744073709551516], entry
[syscall] id = 79, args = [18446744073709551516, 8032, 1073740144, 256, 8032, 18446744073709551516], return 0

...... 省略若干条 FSTATAT

[syscall] id = GETDENTS64, args = [3, 4344, 2048, 2022, 2022, 4320], entry
[syscall] id = 61, args = [3, 4344, 2048, 2022, 2022, 4320], return 0
[syscall] id = CLOSE, args = [3, 0, 0, 0, 0, 0], entry
[syscall] id = 57, args = [3, 0, 0, 0, 0, 0], return 0
[syscall] id = IOCTL, args = [1, 21523, 1073740360, 1, 1073740399, 1454544], entry
[syscall] id = 29, args = [1, 21523, 1073740360, 1, 1073740399, 1454544], return 0
[syscall] id = WRITEV, args = [1, 1073740272, 2, 1, 1073740399, 1461432], entry
arithoh                 fstime                  max_min.lua
[syscall] id = 66, args = [1, 1073740272, 2, 1, 1073740399, 1461432], return 87

...... 省略若干条 WRITEV，这里是 ls 对终端的输出

[syscall] id = EXIT_GROUP, args = [0, 0, 0, 1459768, 0, 0], entry
```

[上一节如何使用往届内核](./justrun.md#注意事项)提到，当我们不知道一个功能如何实现时，可以先看看往届内核的 `syscall` 输出进行对拍。但如果我们要调试往届内核的错误呢？答案是找 Linux 的 `syscall` 输出对拍。
