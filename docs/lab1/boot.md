## 扩展阅读：RISC-V 架构与内核启动

其实 [`rCore-Tutorial` 指导书第一章](http://learningos.cn/rCore-Tutorial-Guide-2023A/chapter1/4mini-rt-baremetal.html) 已经对 `rCore-Tutorial` 的内核启动过程做了大致描述，我们仅在此简要描述流程，并且补充必要的内容。[全国大学生计算机系统能力大赛操作系统设计赛](https://os.educg.net/#/) 内核赛道要求采用 `RISC-V` 架构，所以我们在这里会简单解释 `RISC-V` 的内容。如果你希望开发 `x86_64` 或者 `arm` 的内核，可以参考支持这项架构的内核。例如[Arceos](https://github.com/rcore-os/arceos/) 是一个支持上述这些架构的内核，感兴趣的同学可以去看看它的启动过程。

### RISC-V 特权级

在 RISC-V 架构上有不同的特权级（`priviledge level`），每个特权级有不同的权限。在没有虚拟化的场景下，权限从高到低分别是

- `Machine` 模式（或称 M 态）。这个模式可以访问所有资源，拥有所有权限。默认情况下，所有的异常中断处理都是在 M 态进行的，只有通过它的“允许”（修改一些只有 M 态才能访问的寄存器），部分的异常中断才能被委托给操作系统处理
- `Supervisor` 模式（或称 S 态）。操作系统内核通常运行操作系统内核，内核需要 M 态的程序为它准备一些执行环境，也可以调用 M 态的程序完成一些任务。例如 `rCore-Tutorial` 中内核向串口输出就是向运行在 M 态的 `Rustsbi` 发送 `sbicall` 请求实现的。
- `User` 模式（或称 U 态），一般用于执行真正的用户程序，权限最低。U 态的程序一般不能访问任何硬件资源，只能通过调用 `syscall` 请求 S 态的操作系统处理。当然，也有许多研究通过让 U 态的用户程序直接处理硬件驱动的方式来提高执行效率，那就是另一个故事了。

高权限的模式可以访问低权限模式的一切信息，并控制地权限模式的执行流程。系统启动流程也是从高权限模式开始，初始化完成后再启动低特权级的模式。

有虚拟化的情况下，还会有类似 S 态的 `hypervisor` 模式（HS 态），和 S 态权限一致，但它可以作为一个虚拟的 M 态，在上面支持多个操作系统，是一种“虚拟机”。在 HS 态之上的虚拟操作系统运行在 VS 态，在这样的操作系统上运行的用户程序运行在 VU 态。本课程不涉及虚拟化的知识，如果同学对这一部分内容感兴趣，可以在 [`LearnOS`` 课程首页](https://github.com/LearningOS/) 寻找 `RVM` 相关内容学习，例如 [`RVM-Tutorial`](https://github.com/LearningOS/RVM-Tutorial)

每台 RISC-V 机器并不一定需要具有所有的特权级。例如在嵌入式设备上只需要 M 态，所有的程序都直接运行在 M 态。我们这个课程讲的是操作系统，在运行时通常包含 M/S/U 三个特权级。

### Qemu 启动

`Qemu` 是一个模拟器，为了不把大家绕晕，这里不具体介绍 Qemu 的实现了，只需要假设 `Qemu` 启动后我们就有了一台 `RISC-V` 架构的机器就行。通过 `os/Makefile` 里给 Qemu 的启动参数，可以了解到我们这台“机器”都有什么：

> 不要在终端里输入下面的代码，它不是用来直接执行的，而是写在 `Makefile` 里的

```makefile
qemu-system-riscv64 \
            -machine virt \
            -nographic \
            -bios $(BOOTLOADER) \
            -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
```

- 其中 `-bios` 制定了一个 BootLoader，它是运行在 M 态的程序，负责初始化以及启动内核，默认加载到内存中 `0x80000000` 这个地址。默认情况下， `Qemu` 会使用 `OpenSBI` 启动，但 `rCore-Tutorial` 在这里指定了使用 [`RustSBI`](https://github.com/rustsbi/rustsbi-qemu/) 启动（ `$(BOOTLOADER)` 这个常量的值见 `os/Makefile` 开头处），它是首个使用 Rust 编写的 SBI 实现。
- `-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)` 指定了内核加载的位置是 `KERNEL_ENTRY_PA`，即 `0x80200000`，内核镜像会被放置在内存的这个位置。

### 启动过程

在内核启动时，经历了以下步骤：

1. CPU 从物理地址 `0x1000` 开始执行一段硬件（在我们的实验中是 `Qemu` 模拟出的）上的引导代码，此时 CPU 位于 M 态。

2. CPU 跳转到 `0x80000000` 执行 `RustSBI` 的初始化代码，此时 CPU 位于 M 态。之后通过 `mret` 跳转到 `0x80200000` 执行内核的第一条代码，同时 CPU 切换到 S 态。
   
   1. 其实内核中也有一条类似的指令，就是 `os/src/trap/trap.S` 中的 `sret`，它会在跳转的同时从 S 态切换到 U 态。
   2. `0x80200000` 这个地址是直接编码在 `SBI` 里的，不需要 `Qemu` 告诉它内核在哪。

3. 内核的第一条代码是位于 `os/src/entry.asm` 中的 `_start` 符号所在的位置。这个文件包含以下内容
   
   ```asm6502
       .section .text.entry
       .globl _start
   _start:
       la sp, boot_stack_top
       call rust_main
   
       .section .bss.stack
       .globl boot_stack_lower_bound
   boot_stack_lower_bound:
       .space 4096 * 16
       .globl boot_stack_top
   boot_stack_top:
   ```
   
    它在 `.bss` 段预留了 `16*4K` 的空间作为启动栈。内核启动后，首先通过 `la sp, boot_stack_top` 将这个启动栈的位置写入 `sp` 寄存器，然后通过 `call rust_main` 跳转到 `os/src/main.rs:rust_main` 执行。这是因为高级语言里函数调用需要使用栈帧，所以需要在最开始的汇编代码中初始化 `sp`。

4. 随后就是 Rust 代码控制的启动流程了：

```rust
pub fn rust_main() -> ! {
    clear_bss();
    println!("[kernel] Hello, world!");
    logging::init();
    mm::init();
    mm::remap_test();
    trap::init();
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    fs::list_apps();
    task::add_initproc();
    task::run_tasks();
    panic!("Unreachable in rust_main!");
}
```

这些函数实际上包含了以下启动的步骤：

1. `clear_bss` 把 `.bss` 段清零，这里通常存放的是全局变量。
2. `logging::init()` 初始化了日志，之后就可以使用 `error!` `info!` 等进行输出了，而不只是 `println!`
3. `mm::init()` 初始化了内核堆、页帧分配器和内核页表。内核堆初始化后，可以使用 `Vec` 等动态大小的结构；页帧分配器初始化后，可以给页表分配页面；内核页表初始化后，可以给内核各段限制权限了，例如代码段 `.text` 是只读的，但在此之前可以任意修改内核中任意地址处的值。
4. `trap::init()` 初始化了异常中断处理过程。如果在此之前发生异常、中断，会直接由 M 态处理，通常会直接导致内核退出。
5. `trap::enable_timer_interrupt()` 和 `timer::set_next_trigger()` 开启了时钟中断，此后每隔一段时间，异常中断处理模块 `os/src/trap/` 就会收到一次时钟中断。不过目前的 `rCore-Tutorial` 在内核态时会屏蔽中断，因此只有通过 `sret` 进入用户态后它才会触发。
6. `fs::list_apps()` `task::add_initproc()` 访问文件系统并将用户程序加载到内核
7. `task::run_tasks()` 开始进入第一个用户程序并执行
