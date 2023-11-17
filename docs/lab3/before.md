# 部分往届内核及运行指引

除了这次试验要介绍的 `Starry` 之外，我们还推荐参考以下内核的实现：

| 名字和链接                                                                                     | 学校          | 年份   | 使用语言 | 部分特性                                               |
| ----------------------------------------------------------------------------------------- | ----------- | ---- | ---- | -------------------------------------------------- |
| [umi](https://github.com/js2xxx/umi/)                                                     | 西安交通大学      | 2023 | Rust | 中断驱动的全局异步；函数自动转换为 future；模块化；用户地址检查；基于smoltcp 的协议栈 |
| [Titanix](https://github.com/greenhandzpx/Titanix)                                        | 哈尔滨工业大学（深圳） | 2023 | Rust | 部分异步；用户地址检查；支持 redis、gcc、vi 和一个 httpserver         |
| [ByteOS](https://github.com/yfblock/ByteOS)                                               | 河南科技大学      | 2023 | Rust | 异步函数包装；支持 redis、gcc、ssh 和一个 httpserver             |
| [alien](https://gitlab.eduxiji.net/202310007101563/Alien)                                 | 北京理工大学      | 2023 | Rust | 支持 GUI；支持 bash、redis、sqlite3                       |
| [FarmOS](https://gitlab.eduxiji.net/202310006101080/zhongtianos)                          | 北京航空航天大学    | 2023 | C    |                                                    |
| [FTLOS](https://gitlab.eduxiji.net/DarkAngelEX/oskernel2022-ftlos)                        | 哈尔滨工业大学（深圳） | 2022 | Rust | 异步函数包装；RCU；syscall 快速路径                            |
| [maturin](https://github.com/scPointer/maturin)                                           | 清华大学        | 2022 | Rust | cargo生成文档；模块化；支持 redis、gcc                         |
| [Oops](https://gitlab.eduxiji.net/educg-group-14158-894147/oskernel2022-oops)             | 哈尔滨工业大学（深圳） | 2022 | Rust | 支持 binutils、vi、rustc                               |
| [TatakaOS](https://gitlab.eduxiji.net/YzTz/os)                                            | 杭州电子科技大学    | 2022 | C    | 支持 vi、quickjs、xc（一个C的解释器）                          |
| [NPUcore](https://github.com/echodjx/NPUcore)                                             | 哈尔滨工业大学（深圳） | 2022 | Rust | 带缓存的VFS；支持 bash                                    |
| [TLTS](https://gitlab.eduxiji.net/educg-group-14239-914332/oskernel2022-x)                | 北京航空航天大学    | 2022 | C    | 支持 redis、gcc                                       |
| [UltraOS](https://github.com/oscomp/2021oscomp-best-kernel-design-impl/tree/2021-ultraos) | 哈尔滨工业大学（深圳） | 2021 | Rust | 运行时调试工具                                            |
| [404](https://github.com/oscomp/2021oscomp-best-kernel-design-impl/tree/2021-404)         | 中国科学院大学     | 2021 | C    |                                                    |
| [xv6](https://github.com/oscomp/2021oscomp-best-kernel-design-impl/tree/2021-xv6)         | 华中科技大学      | 2021 | C    | 多核启动                                               |

> 上表中没有提及比赛要求实现的测例集和应用，只列出了要求之外的部分。另外，对于内核特性的描述比较主观。如果你有不同的想法，可以在[本项目的 `issue`](https://github.com/scPointer/rcore2oscomp/issues) 告诉我们。

在 2021 年的第一届比赛时，还没有队伍能够完成赛事方提供的所有测例（主要是 `lmbench` 测例集），所以内核本身的特色会比较少。而在 2022 年的第二届比赛，有很多队伍完成了所有测例，并开始尝试支持更大的应用程序。到了 2023 年的第三届比赛，参赛队伍不仅在自己的内核上运行比赛没有要求的应用程序，还支持了许多现代内核特性。

表中的大部分内核都是用 Rust 写的。如果你找得足够仔细的话，这些 Rust 内核都在文档里鸣谢了 `rCore-Tutorial` 或者直接使用了它的的代码，因此学习 `rCore-Tutorial` 本身会对你理解他们的代码有很大的帮助！当然，每一届的内核也会参照往届其他内核的实现。在完成自己的内核时，不要忘了尝试运行一下这些优秀的内核，这会帮你节省很多 debug 的时间。

## 运行注意事项

由于比赛包含一些未写明的机制、评测机环境、测例约定，所以尝试运行它们可能会遇到一些问题

- 如果是 2022 或 2023 年的作品，且赛后在 `github` 上有维护，那么大概率可以直接运行，不会和比赛环境有太大联系。本实验要用的 [Starry](https://github.com/Azure-stars/Starry/) 也是这种模式，且目前仍然在持续更新。
- 如果是来自 `gitlab` 的链接，那么有可能这个内核只能在比赛环境运行，我们想在赛后运行它可能要花一些功夫。

具体来说，可能有以下问题

### Rust 编译器版本

目前所有比赛中的 Rust 内核以及 `rCore-Tutorial` 都是用 `rust-nightly` 版本。在项目的根目录下，运行 `cargo --version` 即可查看项目使用的 Rust 版本。**`nightly` 的更新非常频繁，包含许多不稳定的特性，而这些特性随时可能改动。所以往年比赛时用的 `nightly` 版本现在可能已经不再适用**。

这个版本记录在项目的根目录下的 `rust-toolchain.toml` 或者 `rust-toolchain` 文件中。如果没有这个文件，则会使用默认版本（一般是本机的架构的稳定版，`x86_64` 或 `arm64`）。例如 [`Starry` 的 `rust-toolchain.toml`](https://github.com/Azure-stars/Starry/blob/main/rust-toolchain.toml) 写的是

```
[toolchain]
profile = "minimal"
channel = "nightly"
components = ["rust-src", "llvm-tools-preview", "rustfmt", "clippy"]
targets = ["x86_64-unknown-none", "riscv64gc-unknown-none-elf", "aarch64-unknown-none-softfloat"]
```

`channel = "nightly"` 意味着这个项目会主动跟随最新的 `cargo` 版本。这个内核在持续开发中，会及时修正代码中的版本冲突。

而 [`NPUcore` 的 `rust-toolchain`](https://github.com/echodjx/NPUcore/blob/main/NPUcore_2022/rust-toolchain) 写的是

```
nightly-2022-04-11
```

这是当年比赛要求使用的版本。直接尝试运行这个内核不会有太大问题，Rust 会自动下载 `nightly-2022-04-11` 这个版本。但如果你想在这个内核上继续开发，那么为了避免和新的库冲突，最好还是把它换成最新的版本，然后修改必要的特性和代码。

还有一种情况是 [`Oops`](https://gitlab.eduxiji.net/educg-group-14158-894147/oskernel2022-oops)，此时内核没有指定  `rust-toolchain.toml` 或者 `rust-toolchain` ，因此你在运行它时会默认采用最新的 `cargo`。但这个内核开发时的环境（指那一年评测机环境和队员的本地环境）其实是 `nightly-2022-04-11`，所以直接运行时可能会报错。给它加上一个和上面提到的 `NPUcore` 同样的版本文件可以规避掉这个问题。

### 评测机和离线编译

平时我们联网编译内核时，包管理器兼编译器 `cargo` 会根据 `Cargo.toml` 文件在包管理平台或者指定的 `git` 仓库下载到本地的 `~/.cargo` 目录下作为缓存（除非依赖指定的是本地的路径），然后再进行编译。但在指定离线编译的情况下，包管理器只能找到找本地 `~/.cargo` 里有的库和用相对路径指定的库。

至少从前三届的比赛来看，评测机是不联网的。评测机的 `~/.cargo` 目录自带一些常用的库，这些库通常会在评测页面列出。但如果内核要用其他的库，就只能把它“复制粘贴”到自己的项目里了。以 `FTLOS` 为例，[它的 `Cargo.toml`](https://gitlab.eduxiji.net/DarkAngelEX/oskernel2022-ftlos/-/blob/master/code/kernel/Cargo.toml) 里就有一些网上开源的库，但因为评测机上没有，所以只能复制到 [`../dependencies` 目录](https://gitlab.eduxiji.net/DarkAngelEX/oskernel2022-ftlos/-/tree/master/code/dependencies)下使用了。

当然，引发问题的通常不是这些库，而是**评测机本地有，但你的本地没有**的库。由于指定了离线编译，所以编译器只会在 `~/.cargo` 下试图找这些库，找不到时就会报错。此时只要把离线编译关掉就好了。离线编译是通过 `--offline` 形式指定的。一般来说，如果你看到内核目录的 `Makefile` 里有

```
cargo build --offline
```

那么把 `--offline` 删掉就可以了，这会迫使 `cargo` 联网去找这些库。比如 [ByteOS 的实现](https://github.com/yfblock/ByteOS/blob/main/Makefile#L44) 就是如此。如果内核 `Makefile` 写得比较花，也可能用其他方式传参，比如 [FTLOS 的实现](https://gitlab.eduxiji.net/DarkAngelEX/oskernel2022-ftlos/-/blob/master/makefile#L5) 就是藏在根目录下。你可以全局搜索 `--offline` 来找这个选项。

### 默认运行的文件

在比赛中，内核的编译使用的是每个队伍自己代码仓库里的 `Makefile`，但加载和运行会使用评测机上的脚本。

以大家熟悉的 `rCore-Tutorial` 为例，`make run` 时会先编译出一个内核镜像 `os.bin`，然后这个内核镜像与 `RustSBI`、文件系统镜像一起作为参数被扔给 `Qemu` 运行。对应到比赛中，就是评测机先调用项目根目录下 `make all` 得到 `os.bin`，然后再用一套自己的脚本去配置实机或 `Qemu` 并运行。这套脚本与每个队伍代码仓库中的运行方式无关，所以如果你的配置和它不太一样，就可能无法正确运行。例如 2022 年比赛中，评测机会将官方指定的文件系统镜像直接映射到物理内存的 `[0x9000_0000, cfff_ffff)`，有一些参赛队会默认按这个逻辑实现。

不过，本节开头列出的内核都可以不依赖评测机环境、根据 `README` 指引生成文件系统镜像并运行。所以只有在运行其他参赛内核时你才需要担心这个问题。

