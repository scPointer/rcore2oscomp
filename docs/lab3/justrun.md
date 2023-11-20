# 如何使用往届内核

[`Starry`](https://github.com/Azure-stars/Starry/) 是 2023年比赛的参赛作品，基于 [`Arceos`](https://github.com/rcore-os/arceos/) 实现，目前仍在更新，运行它几乎不会遇到 [部分往届内核及运行指引](./before.md) 一节中提到的问题。

我们会带大家从一个完全陌生的视角了解 `Starry`，**你可以用同样的方式去运行和理解其他内核！**。

如果需要引入一些额外的知识，我们会用如下的格式来解释：

> `Starry` 只要求安装有 `nightly` 版本的 `rust`，以及 `qemu-system-riscv64`。如果你想用它运行 C 语言编译的测例，还需要在第一个实验中安装的 `riscv64-linux-musl-` 工具链。

## 运行 Starry

看根目录下 `README.md`，在终端中运行

```bash
./build_img.sh sdcard
make run
```

如果顺利的话，会先看到 `OpenSBI` 的 logo 和一堆参数，接下来是 `Arceos` 的内核输出：

```
       d8888                            .d88888b.   .d8888b.
      d88888                           d88P" "Y88b d88P  Y88b
     d88P888                           888     888 Y88b.
    d88P 888 888d888  .d8888b  .d88b.  888     888  "Y888b.
   d88P  888 888P"   d88P"    d8P  Y8b 888     888     "Y88b.
  d88P   888 888     888      88888888 888     888       "888
 d8888888888 888     Y88b.    Y8b.     Y88b. .d88P Y88b  d88P
d88P     888 888      "Y8888P  "Y8888   "Y88888P"   "Y8888P"

arch = riscv64
platform = riscv64-qemu-virt
target = riscv64gc-unknown-none-elf
smp = 1
build_mode = release
log_level = warn

[  3.976575 0 fatfs::dir:140] Is a directory
[  8.753755 0 fatfs::dir:140] Is a directory
[ 13.505061 0 fatfs::dir:140] Is a directory
[ 18.393231 0 fatfs::dir:140] Is a directory
[ 23.440344 0 fatfs::dir:140] Is a directory
[ 62.351342 0:8 axprocess::signal:128] cpu: 0, task: 8, handler signal: 11
[ 62.365045 0:8 axprocess::signal:90] Terminate process: 7
[ 62.381378 0:6 axprocess::signal:128] cpu: 0, task: 6, handler signal: 17
Simple syscall: 27.7772 microseconds
```

其余的输出还有很长，看来运行还需要一段时间，此时可以不等它执行完所有测例，直接关掉 `Qemu`（按 `ctrl+A` ，松开后按 `X`）。

## 替换测例

面对一个全新的内核，我们可以提出许多问题：

- 它是如何启动的？
- 我们熟悉的内核（比如 `rCore-Tutorial`）里的内存管理、进程、文件系统、异常中断等等功能它也有吗？有什么区别？
- 它有没有自己的规范，或者说代码风格？

但首先应该考虑的是**如何替换它运行的测例以便为我们所用**。以上一章实验中的 `hellostd` 测例为例，如果我们能让 `Starry` 运行这个测例，看一看它执行了哪些 `syscall` ，再找一找这些 `syscall` 的实现，就可以轻松秒杀前一章的实验了。

比赛中涉及的大部分内核功能都可以以这种方式实现，只要你的进度还没有超过往届的内核，就可以借鉴它们的写法去通过同样的测例。

### 修改文件系统镜像

我们的目标是用 `Starry` 运行上一章实验的 `hellostd`。

首先观察前面运行时的命令，`./build_img.sh sdcard` 应该就是制作镜像的步骤了，它指向根目录下的文件 `./build_img.sh`：

```
rm disk.img
dd if=/dev/zero of=disk.img bs=3M count=24
mkfs.vfat -F 32 disk.img
mkdir -p mnt
sudo mount disk.img mnt
# 根据命令行参数生成对应的测例
sudo cp -r ./testcases/$1/* ./mnt/
sudo umount mnt
rm -rf mnt
sudo chmod 777 disk.img
```

这段脚本生成了一个空的镜像文件 `disk.img`，然后把它挂载到 `/mnt`，并将所有在 `./testcases/$1/*` 下的文件拷贝到了镜像中。我们其实在[第一章实验的介绍](../lab1/intro.md#在实验之后) 中介绍过 FAT32 镜像的生成与挂载。其中 `$1` 是执行命令 `./build_img.sh sdcard` 时的第一个参数，也就是 `sdcard`。所以被加载到文件系统里的实际上是项目根目录下的 `testcases/sdcard/` 里的所有文件。

参考[第二章实验的介绍](../lab2/intro.md) 把 `hellostd` 这个测例编译出来(在 `rCore-Tutorial` 那边的 `testcases/build/`下)，然后复制到 `Starry` 的  `testcases/sdcard/` 文件夹下。然后再运行一次 `./build_img.sh sdcard` ，这个测例就在 `Starry` 的文件系统镜像里了。

### 寻找运行测例的位置

现在我们想让 `Starry` 启动时不要运行目前的测例，而是启动 `hellostd`。

每个内核启动后运行的程序都不太一样，例如 `rCore-Tutorial ch8` 开机启动的 `ch8b_initproc` 就藏在 `os/src/task/mod.rs` 的第一百多行。对于一个陌生的内核（不只是 `Starry`），通常有以下方式可以找到它启动后执行什么测例：

- 先看 `README.md`。
  - 不过 `Starry` 在这里并没有相关说明。
- 在具体文档中查找。在 `Starry` 是 `/doc` 目录，在其他内核里可能是类似名字的目录，或者 `README` 中链接到的外部文档。类比 `rCore-Tutorial`，我们需要搜的是“测例”“启动”“第一个进程”之类的说法。
  - 对于 `Starry` 来说，可以在 `doc/OSCOMP-Repo/启动流程.md` 这个文档找到关于 `ulib/starry_libax/src/test.rs/run_testcases.rs` 的描述。虽然这个路径不在内核中，但只要全局搜索 `run_testcases` 就可以找到它现在在哪了。
- 如果你知道这个内核启动时默认在执行什么，也可以直接搜索用户程序的名称。例如启动后出现终端，那很可能是 `busybox sh`。
  - 对于 `Starry` 来说，如果你知道它启动后执行的是来自 `lmbench` 的测例，就可以全局搜索 `lmbench`。搜出文件稍微有点多，但找两分钟还是能找到谁在控制执行测例的。
- 否则，可以搜索 `_start`，找找内核启动的位置，然后找到内核初始化的函数，看第一个用户程序是从哪加载的
  - 对于 `Starry` 来说，会找到 `_start` 跳转 `rust_entry` 再到 `rust_main` 最后是 `main`。全词搜索 `main()` 可以发现几乎都在 `apps/*` 目录下，最像比赛要求的就是 `apps/oscomp/src/main.rs` 了。

条条大路通罗马，你只需要用**其中一种**方式找到启动后执行的测例在哪就行。

**思考题2**：在[部分往届内核及运行指引](./before.md) 一节提到的内核中挑选一个，描述它在默认情况下启动后会执行哪些测例（抑或是直接启动终端）。你不一定要真的运行那个内核，读文档或搜索即可。

### 替换测例并输出 `syscall` 日志

通过上面的方法，我们可以找到 `Starry` 下的 `ulib/axstarry/syscall_entry/src/test.rs` 中的 `run_testcases` 函数：

```rust
pub fn run_testcases(case: &'static str) {
    fs_init(case);
    let (mut test_iter, case_len) = match case {
        "junior" => (Box::new(JUNIOR_TESTCASES.iter()), JUNIOR_TESTCASES.len()),
        "libc-static" => (
            Box::new(LIBC_STATIC_TESTCASES.iter()),
            LIBC_STATIC_TESTCASES.len(),
        ),
        "libc-dynamic" => (
            Box::new(LIBC_DYNAMIC_TESTCASES.iter()),
            LIBC_DYNAMIC_TESTCASES.len(),
        ),
        "lua" => (Box::new(LUA_TESTCASES.iter()), LUA_TESTCASES.len()),
        "netperf" => (Box::new(NETPERF_TESTCASES.iter()), NETPERF_TESTCASES.len()),

        "ipref" => (Box::new(IPERF_TESTCASES.iter()), IPERF_TESTCASES.len()),

        "sdcard" => (Box::new(SDCARD_TESTCASES.iter()), SDCARD_TESTCASES.len()),

        "ostrain" => (Box::new(OSTRAIN_TESTCASES.iter()), OSTRAIN_TESTCASES.len()),
        _ => {
            panic!("unknown test case: {}", case);
        }
    };
    ......
```

回顾我们最初编译文件系统镜像，使用的是 `./build_img.sh sdcard`，那应该就是 `SDCARD_TESTCASES` 这个常量了。在同个源文件下找到它，直接删掉所有项目，改成

```rust
pub const SDCARD_TESTCASES: &[&str] = &[
    "hellostd",
];
```

再次 `make run` 就可以运行我们自己塞进去的测例了。不过 `Starry` 可以通过比赛要求的所有决赛测例，因此它可以运行 `hellostd` 这件事没有什么特别的。我们真正想要知道的是它调用了什么 `syscall`。

找到系统调用的入口，也就是 `ulib/axstarry/syscall_entry/src/syscall.rs` 的函数 `syscall`，然后在开头和结尾各加一句 `println!` 输出。

> 一般来说，通过文档或者全局搜索 `syscall(`、`sys_` 都能很容易找到一个内核的系统调用模块。有把启动过程藏得特别深的内核，也有把初始进程藏到配置文件的内核，但处理 `syscall` 的模块肯定都叫 `syscall`。

```rust
#[no_mangle]
pub fn syscall(syscall_id: usize, args: [usize; 6]) -> isize {
    println!("[syscall] id = {}, args = {:?}, entry", syscall_id, args);
    #[cfg(feature = "futex")]
    syscall_task::check_dead_wait();
    ......
    let ans = deal_result(ans);
    ans
    println!("[syscall] id = {}, args = {:?}, return {}", syscall_id, args, ans);
}
```

然后 `make run` 会得到报错：

```bash
error: cannot find macro `println` in this scope
 --> ulib/axstarry/syscall_entry/src/syscall.rs:5:5
  |
5 |     println!("[syscall] id = {}, args = {:?}, entry", syscall_id, args);
  |     ^^^^^^^

error: cannot find macro `println` in this scope
  --> ulib/axstarry/syscall_entry/src/syscall.rs:42:5
   |
42 |     println!("[syscall] id = {}, args = {:?}, return {}", syscall_id, args, ans);
   |  
```

对于大部分的内核来说，直接输出 `println!` 是可以的。但因为 `Starry` 是一个模块化程度比较高的内核，所以它的 `syscall` 模块对于内核来说就像是一个普通的“依赖库”一样（还记得[上一节](./cache.md) 中我们说不能在依赖库中 `println` 吗），所以不支持 `println!`。不过此时我们可以试一试用 `error!` `warn!` `info!` 这些命令，因为 `rCore-Tutorial` 使用的日志库也是一个非常流行的库。把代码修改成

```rust
#[no_mangle]
pub fn syscall(syscall_id: usize, args: [usize; 6]) -> isize {
    error!("[syscall] id = {}, args = {:?}, entry", syscall_id, args);
    #[cfg(feature = "futex")]
    syscall_task::check_dead_wait();
    ......
    let ans = deal_result(ans);
    ans
    error!("[syscall] id = {}, args = {:?}, return {}", syscall_id, args, ans);
}
```

再次 `make run` 后报错变成了

```
error: cannot find macro `error` in this scope
 --> ulib/axstarry/syscall_entry/src/syscall.rs:5:5
  |
5 |     error!("[syscall] id = {}, args = {:?}, entry", syscall_id, args);
  |     ^^^^^
  |
help: consider importing this macro
  |
1 + use axlog::error;
  |

error: cannot find macro `error` in this scope
  --> ulib/axstarry/syscall_entry/src/syscall.rs:42:5
   |
42 |     error!("[syscall] id = {}, args = {:?}, return {}", syscall_id, args, ans);
   |     ^^^^^
   |
help: consider importing this macro
   |
1  + use axlog::error;
   |
```

编译器提示说可以加上 `use axlog::error;`。我们在 `syscall()` 函数前面开始前、 `use syscall_utils::deal_result;` 之后加上一行声明 `use axlog::error;`。再次 `make run`，现在可以成功运行并拿到正确输出了！

>  **说明**：`Starry` 的默认调试输出级别是 `warn!` 及以上，所以这里用 `error!` 可以看到输出。它是在根目录下 `Makefile` 的 `LOG ?= warn` 这一句设定的。
> 
> 如果在运行其他内核时没有看到调试输出，可以找找它的 `Log` 设置，一般都和 `rCore-Tutorial` 差不多。也有个别内核自己写了一套调试输出方法，不过这样做的队伍一定会在 `README.md` 告诉你怎么调试的。

为了不给还没做完 `lab2` 先来看 `lab3` 实验的同学剧透，这里就不放具体输出内容了。

## 基于 `syscall` 的对拍调试以及思考

上面这个方法具体有什么用？知道了测例对应的 `syscall` 参数和输出，就可以仿照 `Starry` 去写对应的 `syscall`，它返回 0 的你也可以返回 0；它使用其他 `syscall` 代替的你也可以用其他 `syscall` 代替。这就相当于你有了一份关于 `lab2` 的参考答案。如果只是看代码，不加 `syscall` 输出，就可能需要仔细阅读 `Starry` 的源码才能知道它需要哪些实现。

更进一步，既然我们可以让 `Starry` 运行 `hellostd`，自然也能让它运行所有的决赛测例。这样一来，只要你的进度还在比赛要求测例的范围内，就基本不需要担心在同一个测例上卡太久。出现问题时，可以对比往届内核给出的 `syscall` 参数和返回值和自己内核给出的 `syscall` 参数和返回值，基本都能快速定位到问题。这种调试方式叫做对拍，如果还有印象的话，我们在[第一章的测例库介绍一节](../lab1/clib.md)的末尾介绍过它。

当然，如何比以前的内核写得效率更高、更模块化、支持更多的功能就是另一个话题了。

> 大部分初赛测例也可以运行，但因为[初赛的 `syscall` 规范](https://github.com/oscomp/testsuits-for-oskernel/blob/main/oscomp_syscalls.md)并不是真正的 `POSIX syscall`，而是做了一定修改，所以不一定能在完成决赛要求的内核上运行。做到决赛的内核都是需要满足 `POSIX syscall` 规范的。
> 
> 通过 `syscall` 输出调试可以解决 80% 的问题，但并不是全部问题。有可能你的内核和往届内核在 `syscall` 之外的模块有不同，导致只看 `syscall` 的输出无法完成调试，例如页表和地址空间设置有误，或是没有保存用户程序的浮点寄存器等等。

### 注意事项

在使用 `syscall` 输出调试时，还有其他需要注意的地方：

- 一些 `syscall` 包含用户空间的数据结构，例如 `sys_clock_gettime` 就会传一个 `TimeSpec` 用于获取时间（你可能在 [`rCore-Tutorial ch4`](http://learningos.cn/rCore-Tutorial-Guide-2023A/chapter4/7exercise.html) 重写过类似的 `syscall`）。这时候仅输出地址可能不足以帮助调试，最好还是进到 `sys_clock_gettime` 里去给它“定制”一句调试输出。
- 后期的 `lmbench` 等测例集会反复调用同一个 `syscall` 几十万甚至百万次，以便测试性能。所以在调试这些测例时，最好不要每次 `syscall` 都输出一行，否则屏幕很快就会被输出淹没。万一出现这种情况，记得直接关掉 `Qemu`（按 `ctrl+A` ，松开后按 `X`）。
  - 这个问题有一些很暴力的解决方式。比如可以只输出前几十次调用，也可以用一个类似 [`rCore-Tutorial ch3` 的 task_info](http://learningos.cn/rCore-Tutorial-Guide-2023A/chapter3/5exercise.html) 的全局计数器，然后同个 `syscall` 每隔几百几千次调用才输出一次。
- 有些应用会在没事干的时候反复调用几个 `syscall` 检查是否有输入信息，或者更新系统时间，例如 `redis`。在调试这些应用的时候，可以直接用 `if` 判断特定的 `syscall_id` ，然后不输出这些 `syscall` 的调用信息。

另外，上面在修改 `Starry` 的代码时添加的语句可能会引起你的好奇：

- 为什么要用 `error!`，我们只是想拿到一些内核运行的信息，用 `info!` 或者 `debug!` 是不是更合适？
  - 这是因为内核里可能自带很多 `Log`  输出，如果我们想要的输出等级不够高，可能就会淹没在内核本身的日志输出里，亦或是因为 `Log` 等级太低根本看不到输出。当然，如果可以的话直接敲 `println!` 还是更方便一些。
- 为什么没有添加其他信息，比如调用这个 `syscall` 的进程和线程 ID？
  - 是的，完全可以增加这些信息！这一节的内容只是帮助大家以最短的时间去把往届内核用起来，如果你找到了内核里哪里存进程和线程 ID，也可以把它放进输出里。

>  **说明**：顺便一提，其实 `Starry` 的调试输出里有线程 ID，例如 `[ 41.956958 0:6 syscall_entry::syscall:7]` 指的是在启动后 `41.956958` 这个时间点，CPU `0` 号核心在 `TID=6` 这个线程上，运行到 `syscall_entry::syscall` 的第 `7` 行时打印了输出。
> 
> 不过 `Starry` 没说它是线程 ID，我们需要阅读 `modules/axlog/` 的代码才能知道这些信息

还有一些问题留作思考题：

**思考题3.1**：为什么要在开头结尾各输出一句，会不会太过重复？（提示：考虑执行出错的情况，或者 `sys_exit` ）

**思考题3.2**：为什么要结尾还要输出一遍 `syscall` 的完整参数，只输出返回值行不行？（提示：考虑像 `sys_yield` 这样的 `syscall`）
