# 使用 strace 调试

在本节中，我们会使用 `strace` 分析 `busybox ls` 的输出，并利用这一点找出为什么内核的 `busybox ls` 没有显示所有的目录文件。

本节会大量修改 `Starry` 下 `ulib/axstarry/syscall_fs/src/imp/ctl.rs` 的代码，你不需要跟着修改，可以在[这里](https://github.com/LearningOS/2023a-stage3-proj2/tree/lab3)获取同名的 `ctl.rs` 文件，然后替换掉原本的 `ctl.rs`。

> **注意**：写作本节指导书时，使用的环境是 `Ubuntu 22.04`，`busybox` 的版本是静态链接的 `v1.30.1 (Ubuntu 1:1.30.1-7ubuntu3)`。如果你的环境不太一样，输出可能会略有不同，但原理是相通的。

事实上靠 `strace` 调试 `busybox ls` 比较难，反而是调试本章实验要求做的 `busybox mv` 要更容易一些，所以你可能会觉得本节的调试过程有一点别扭和繁琐。但正因为调 `busybox ls` 更难，所以我们把它放在了指导书里而不是留作作业。


### 尝试在本机上获取 `strace` 输出

你可以按[上一节提到的方法](./strace.md)在 `riscv64-linux` 下测试，会得到更干净的输出。但为了实验简便，我们这里演示直接在本机上测试的思路。

运行 `Starry` 时执行 `ls` 的根目录在 `Starry` 代码目录的 `testcases/sdcsrd/`。我们在本机这个目录下运行 `strace busybox ls` 可以得到以下输出

```bash
...... 省略进程初始化

newfstatat(AT_FDCWD, ".", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
newfstatat(3, "", {st_mode=S_IFDIR|0755, st_size=4096, ...}, AT_EMPTY_PATH) = 0
getdents64(3, 0x14d2850 /* 76 entries */, 32768) = 2632
newfstatat(AT_FDCWD, "./arithoh", {st_mode=S_IFREG|0755, st_size=112544, ...}, AT_SYMLINK_NOFOLLOW) = 0
newfstatat(AT_FDCWD, "./syscall", {st_mode=S_IFREG|0755, st_size=117864, ...}, AT_SYMLINK_NOFOLLOW) = 0

...... 省略若干条 newfstatat

getdents64(3, 0x14d2850 /* 0 entries */, 32768) = 0
close(3)                                = 0
newfstatat(1, "", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x3), ...}, AT_EMPTY_PATH) = 0
write(1, "arithoh                 execl   "..., 78arithoh                 execl                   lmbench_testcode.sh     short
) = 78

...... 省略若干条 write

exit_group(0)                           = ?
```

浏览一下上面各个 `syscall` 的文档，可以找到 `getdents64` 就是获取目录下各个文件名的 `syscall`，它很可能有问题，但在此之前，我们还是先对比一下在 [“定位 `ls` 命令存在的问题”一节](./ls.md) 末尾的给出的内核输出：

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

可以看到还是有些不一样的地方。不过我们可以通过第二章[通过 Manual Page 添加 Syscall](../lab2/syscall.md)一节的方法，去依次检查两个输出不一样的地方，并尝试给出解释：

1. `Starry` 的 `FSTATAT` 和本机的 `newfstatat` 是同一个 `syscall` 吗？
   
是的。在 `Starry` 全局搜索 `FSTATAT` 可知调用号是 `79`。在[`riscv linux syscall`列表](https://jborza.com/post/2021-05-11-riscv-linux-syscalls/) 中可以查到 `79` 号就是 `newfstatat`。

[这个 `syscall` 的文档](https://man7.org/linux/man-pages/man2/newfstatat.2.html)说它是 `stat, fstat, lstat, fstatat - get file status`，是一个获取文件信息的 `syscall`。

2. `Starry` 的是 `FSTATAT` 输出中反复出现 `18446744073709551516`，本机输出中 `newfstatat` 反复出现 `AT_FDCWD`，它们是一样的吗？

是的。在 `Starry` 全局搜索 `AT_FDCWD` 可以找到这个常量的定义是 `-100`，它写成 64 位无符号数就变成了一个输出里的那个数。同样在[`stat`的文档](https://man7.org/linux/man-pages/man2/newfstatat.2.html) 可以搜到这个常量表示的是“当前路径”。

> 比起内核的输出，本机 `strace` 的输出的可读性要高上不少，这是因为我们粗暴地在 `ulib/axstarry/syscall_entry/src/syscall.rs` 里以无符号数的形式输出了每个 `syscall` 的所有参数。如果我们想让内核输出的可读性更强，可以改为在每一个具体的 `syscall` 函数内部添加输出。
>
> 如果一个参数是数字，可以直接用 `println!("{}",a);` 输出；如果参数是地址，那么可以用 `println!("{:x}",a);` 获得十六进制输出；如果参数包含各种符号位（通常用 `bitflags` 依赖库），或者是枚举体（通常用 `numeric-enum-macro` 依赖库），可以用 `println!("{:#?}",a);` 输出。
> 
> `?` 表示将输出交给 `Debug` 方法而不是 `Display` 方法执行。如果内核里定义了新的结构体，也可以 `impl Debug for ...` 来自定义独特的输出方式。

3. `syscall` 调用中还有对不上的地方：使用 `open` 打开当前目录后，本机输出一条文件名为空的 `newfstatat`，内核输出一条 `FCNTL64`；使用 `close` 关闭当前目录后，内核输出了一条 `IOCTL`。是它们影响了输出结果吗？

这些不一致不影响结果。

- 一样可以查 `syscall` 文档得知带 `AT_EMPTY_PATH` 参数且文件名为空时，`newfstatat` 查询的是第一个参数代表的文件标识符指向的目录。在这个例子中是 `3`，也就是上一条 `open` 打开的文件。
- 在 `Starry` 全局搜索 `fcntl64` （最好能对照 `syscall` 文档）可知这条 `syscall` 的含义是对给出的文件标识符 `3` 设置 `F_SETFD`，也即如果发生 `sys_exec`，那么新进程会关闭这个文件标识符，也就是上一条 `open` 打开的文件。后续调用没有 `exec` 所以它没有任何影响。
- 在 `Starry` 全局搜索 `ioctl`，可以找到它的实现。这条控制命令是针对文件描述符 `1` 也就是标准输出的，和目录文件无关。如果再仔细找找，还能找到参数 `21523` 对应常量 `TIOCGWINSZ`，也就是窗口长宽，这就是为什么 `Starry` 中的 `ls` 输出成三列。

4. 内核输出里在一堆 `FSTATAT` 中间有一条 `mmap`，本机输出没有。而且两边的 `getdents64` 的参数有些不太一样。

这里就需要仔细分析了

### 进一步检查 `syscall` 原因和参数

如果你还记得 `rCore-Tutorial ch4` 的话，`mmap` 的作用是申请一块内存映射。查 `mmap` 的 `syscall` 文档可知，第一个参数即起始地址 `addr=0` 时，表示由内核决定一个映射位置（见第二段开头）

```
 mmap() creates a new mapping in the virtual address space of the
       calling process.  The starting address for the new mapping is
       specified in addr.  The length argument specifies the length of
       the mapping (which must be greater than 0).

       If addr is NULL, then the kernel chooses the (page-aligned)
       address at which to create the mapping; this is the most portable
       method of creating a new mapping.
       ......
```

也就是说用户程序只是单纯想多要一块空间存东西，地址由内核随便给。申请新的空间说明肯定是旧的什么地方“不够了”，而 `getdents64` 的作用就是获取目录下各个文件名的 `syscall`，是不是它导致的呢？查 `getdents` 的文档可知：

```
 long syscall(SYS_getdents, unsigned int fd, struct linux_dirent *dirp,
                    unsigned int count);
......

   getdents()
       The system call getdents() reads several linux_dirent structures
       from the directory referred to by the open file descriptor fd
       into the buffer pointed to by dirp.  The argument count specifies
       the size of that buffer.
......
       The getdents64() system call is like getdents(), except that its
       second argument is a pointer to a buffer containing structures of
       the following type:

           struct linux_dirent64 {
               ino64_t        d_ino;    /* 64-bit inode number */
               off64_t        d_off;    /* 64-bit offset to next structure */
               unsigned short d_reclen; /* Size of this dirent */
               unsigned char  d_type;   /* File type */
               char           d_name[]; /* Filename (null-terminated) */
           };
......
RETURN VALUE

       On success, the number of bytes read is returned.  On end of
       directory, 0 is returned.  On error, -1 is returned, and errno is
       set to indicate the error.

```

其中三个参数分别是文件描述符、一组 `linux_dirent64` 结构体地址与 `buffer`（缓冲区）大小，而返回值是获取了多少个 Byte 的信息。对比本机与 `Starry` 的输出

```
# 本机
getdents64(3, 0x14d2850 /* 76 entries */, 32768) = 2632
# Starry 输出
[syscall] id = GETDENTS64, args = [3, 4344, 2048, 0, 0, 4320], entry
[syscall] id = 61, args = [3, 4344, 2048, 0, 0, 4320], return 2022
```

我们终于找到了问题所在：

- 本机的`busybox` 调用时给的 `buffer` 大小是 `32768`，一次获取了 `2632` Byte 的信息就直接输出了；
- `Starry` 下使用比赛官方给的 `riscv64` 下的 `busybox`，调用时给的 `buffer` 大小是 `2048`，一次获取不完，只拿了 `2022` Byte。然后用 `mmap` 申请一次内存，再次调用 `getdents64` 时就没有拿到输出了。

其实这时就可以去尝试修改内核中 `sys_getdentes64` 的实现了，但也可以再想想：在调用前 `busybox` 也不知道目录下有多少文件，所以这个 `buffer` 无论是大是小都是个定值。那如果本机的目录下有超过 `32768` Byte 的信息，`busybox` 是怎么处理的呢？

我们在本机任意位置新建一个目录，然后塞 1500 个空文件进去

```bash
mkdir temp
cd temp
for i in {1..1500}; do touch $i; done
```

然后用 `strace` 查看结果。注意，因为这些文件会触发大量获取文件信息的 `newfstatat` 以及最后输出的 `write`，我们可以利用[strace 介绍一节](./strace.md)中提到的参数，屏蔽这两个 `sycall`。输入命令：

```bash
strace -e trace=\!newfstatat,write busybox ls 
```

除去进程启动和来自 `ls` 命令本身的输出之外，可以看到以下 `syscall` 信息：

```bash
...... 省略进程初始化

openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, 0x250e850 /* 1365 entries */, 32768) = 32760
brk(0x2550000)                          = 0x2550000
getdents64(3, 0x250e850 /* 137 entries */, 32768) = 3288
getdents64(3, 0x250e850 /* 0 entries */, 32768) = 0
close(3)                                = 0
sysinfo({uptime=143511, loads=[3648, 3584, 5408], totalram=16683806720, freeram=8925274112, sharedram=73728, bufferram=323186688, totalswap=4294967296, freeswap=4294967296, procs=621, totalhigh=0, freehigh=0, mem_unit=1}) = 0

...... ls 本身的输出

exit_group(0)                           = ?
```

可以看到一次 `getdents64` 拿不完信息的情况下，本机的 `busybox` 用 `brk` 这个 `syscall` 扩展了自己的数据段大小，然后又调用了两次 `getdents64`。而且每次调用 `getdents64` 时提供的地址都是相同的，但拿到的 `entries` 项不一样。就像是用户程序拿着 `buffer` 问内核“目录下有没有文件”，内核说“这有1365个”；用户又问“还有没有”，内核说“还有137”个；用户又问“还有吗”，内核说“没有了”。

> 我们没有解释为什么 `brk` 和 `mmap` 的作用相同且不影响运行，把它当作已知结论即可。你也可以通过网络搜索得到这个结论。

对比内核输出的两条 `GETDENTS64`：

```
[syscall] id = GETDENTS64, args = [3, 4344, 2048, 0, 0, 4320], entry
[syscall] id = 61, args = [3, 4344, 2048, 0, 0, 4320], return 2022
[syscall] id = GETDENTS64, args = [3, 4344, 2048, 2022, 2022, 4320], entry
[syscall] id = 61, args = [3, 4344, 2048, 2022, 2022, 4320], return 0
```

相当于用户程序拿着 `buffer` 问内核“目录下有没有文件” 的时候把 `buffer` 装满了；但第二次再问时明明还有文件要输出，内核却说“没有了”。看来是第二次 `GETDENTS64` 的返回不对，我们可以由此去修改 `ulib/axstarry/syscall_fs/src/imp/ctl.rs` 中的 `sys_getdents64` 函数。

### 修改代码实现

相比排查错误原因，具体修改代码这部分比较简单，我们会讲的简略一些。

### 检查 `sys_getdents`

我们在 `sys_getdents` 中插入调试输出，看第二次 `GETDENTS64` 是怎么返回 0 的

```rust
/// ulib/axstarry/syscall_fs/src/imp/ctl.rs
pub fn syscall_getdents64(fd: usize, buf: *mut u8, len: usize) -> SyscallResult {
    let path = if let Some(path) = deal_with_path(fd, None, true) {
        path
    } else {
        return Err(SyscallError::EINVAL);
    };

    let process = current_process();
    // 注意是否分配地址
    let start: VirtAddr = (buf as usize).into();
    let end = start + len;
    if process.manual_alloc_range_for_lazy(start, end).is_err() {
        return Err(SyscallError::EFAULT);
    }

    let entry_id_from = unsafe { (*(buf as *const DirEnt)).d_off };
    if entry_id_from == -1 {
        // 说明已经读完了
        error!("is here?");
        return Ok(0);
    }
    ......
    error!("or here?");
    Ok(count as isize)
}
```

可以找到是 `if entry_id_from == -1` 的判断里返回 0，也就是第一个 `DirEnt` 结构体的 `d_off` 一项的值为 `-1` 导致的。它应该是什么？查 `gendents` 文档可知内核里的 `DirEnt` 对应规范里的 `linux_dirent64`

```

long syscall(SYS_getdents, unsigned int fd, struct linux_dirent *dirp,
            unsigned int count);
......
struct linux_dirent64 {
    ino64_t        d_ino;    /* 64-bit inode number */
    off64_t        d_off;    /* 64-bit offset to next structure */
    unsigned short d_reclen; /* Size of this dirent */
    unsigned char  d_type;   /* File type */
    char           d_name[]; /* Filename (null-terminated) */
};
```

也就是说 `d_off` 是这个缓冲区里到下一个结构的偏移。可以看 `syscall` 文档下面的解释，或者我们简单举个例子：假设目录里有三个文件 `a` `b` `c`，存进 `linux_dirent64` 里各占 21 Byte，那么 `a` 放在 `dirp[0]`，`b` 放在 `dirp[21]` ，`c` 放在 `dirp[42]`。所以 `a` 的 `d_off` 是 `21`，`b` 的 `d_off` 是 `42`。

### 修改 `DirEnt` 的实现

现在第二次调用 `GETDENTS64` 时内核里看到的 `d_off` 为 `-1`，这个值是谁给的呢？在 `Starry` 全局搜索 `d_off`，可以找到下面这个结构定义

```rust
/// util/axstarry/syscall_utils/src/ctypes
impl DirEnt {
    /// 定长部分大小
    pub fn fixed_size() -> usize {
        8 + 8 + 2 + 1
    }
    /// 设置定长部分
    pub fn set_fixed_part(&mut self, ino: u64, _off: i64, reclen: usize, type_: DirEntType) {
        self.d_ino = ino;
        self.d_off = -1;
        self.d_reclen = reclen as u16;
        self.d_type = type_ as u8;
    }
}
```

原来罪魁祸首在这里：`sys_getdents` 函数调用了这里的 `set_fixed_part`，导致 `d_off` 的值为 `-1`。

于是我们可以修改这个文件中的目录项的 `d_off: i64` 定义为 `d_off: u64`，然后修改 `set_fixed_part` 函数：

```rust

/// 目录项
#[repr(C)]
#[derive(Clone, Copy)]
pub struct DirEnt {
    /// 索引结点号
    pub d_ino: u64,
    /// 到下一个dirent的偏移
    pub d_off: u64,
    /// 当前dirent的长度
    pub d_reclen: u16,
    /// 文件类型
    pub d_type: u8,
    /// 文件名
    pub d_name: [u8; 0],
}
......
impl DirEnt {
    /// 定长部分大小
    pub fn fixed_size() -> usize {
        8 + 8 + 2 + 1
    }
    /// 设置定长部分
    pub fn set_fixed_part(&mut self, ino: u64, off: u64, reclen: usize, type_: DirEntType) {
        self.d_ino = ino;
        self.d_off = off;
        self.d_reclen = reclen as u16;
        self.d_type = type_ as u8;
    }
}
```

### 修改 `sys_getdents` 的语义和实现

之前这个函数的大致流程是

1. 检查 `d_off` 是否为 `-1`，是则直接返回 0
2. 依次枚举目录项，尝试放进 `DirEnt` （即 `linux_dirent64` 结构）里，并将 `d_off` 参数设置为 `-1`。如果发现用户程序给的 `buffer` 塞不下了，就退出循环
3. 将“上一步向 `buffer` 里一共填了多少 Byte” 作为返回值

我们除了将第二步的每个目录项的 `d_off` 设置成正确语义之外，还要考虑一个问题：下次用户还是拿着同样的 `buffer` 问内核“目录下有没有文件”，但此时 `buffer` 里已经填满了上次内核给的输出，怎么办？

这个问题有很多种解法。我们给出的代码使用这样一种方案：

- 内核填 `buffer` 时，留够空间在最后放一个 `reclen`（即目录项大小）为0，`d_off` 为 `-1` 的空项，类似字符串的 `'\0'`。
- 每次 `sys_getdents64` 先检查这个空项前的所有 `DirEnt`，找它们之中谁的 `d_off` 最大，记作 `all_offset`，这样小于等于这个 `all_offset` 的目录项都可以认为之前已经输出过了。
- 枚举每个目录项，计算如果把它放进 `buffer` 那么 `d_off` 将是多少。如果小于 `all_offset`，就跳过这一项，不实际填进 `buffer`。
- `buffer` 填满时（记得留空间放最后的空项），就退出循环

用上面的流程代替原本的第二步，就可以完成 `sys_getdents` 从而修复 `busybox ls` 命令存在的问题了。修改的代码比较复杂，我们将修改后的代码文件 `ulib/axstarry/syscall_fs/src/imp/ctl.rs` 直接放在了[测例仓库的 `lab3` 分支](https://github.com/LearningOS/2023a-stage3-proj2/tree/lab3)，你可以用它直接替换 `Starry` 中的同名文件。

