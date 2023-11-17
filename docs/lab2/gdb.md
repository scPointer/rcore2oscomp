# GDB 调试

在引入 `loaders` 模块后，我们仍然没有跑通 `hellostd`，这次的运行结果是

```bash
>> hellostd
Shell: Process 2 exited with code -4
```

回顾我们第一次运行这个测例时的输出，可以发现仍然是 `user_shell` 的同一个位置报错（虽然 `user_shell` 的其他地方会 `return -4`，但哪些位置的错误输出都是以 `Error when` 开头，和我们看到的输出不符）。也就是说，这个程序执行失败是在 `user_shell` 在 `waitpid` 等待它执行完成的时候，得知了程序返回 `-4` 的消息。

继续依葫芦画瓢，在 `os/src` 中搜索 `-4`，可以找到 `src/task/signal.rs` 中的 `Some((-4, "Illegal Instruction, SIGILL=4"))`。问题是类似的，但这次我们没有看到 `ERROR` 输出。这是因为 `rCore-Tutorial` 实验框架忘记在 `os/src/trap/mod.rs:trap_handler` 的 `IllegalInstruction` 一项里加错误输出了，我们给它补上：

```rust
......
Trap::Exception(Exception::StoreFault)
| Trap::Exception(Exception::StorePageFault)
| Trap::Exception(Exception::InstructionFault)
| Trap::Exception(Exception::InstructionPageFault)
| Trap::Exception(Exception::LoadFault)
| Trap::Exception(Exception::LoadPageFault) => {
    error!(
        "[kernel] trap_handler: {:?} in application, bad addr = {:#x}, bad instruction = {:#x}, kernel killed it.",
        scause.cause(),
        stval,
        current_trap_cx().sepc,
    );
    current_add_signal(SignalFlags::SIGSEGV);
}
Trap::Exception(Exception::IllegalInstruction) => {
    error!(
        "[kernel] trap_handler: {:?} in application, bad addr = {:#x}, bad instruction = {:#x}, kernel killed it.",
        scause.cause(),
        stval,
        current_trap_cx().sepc,
    );
    current_add_signal(SignalFlags::SIGILL);
}
Trap::Interrupt(Interrupt::SupervisorTimer) => {
    set_next_trigger();
    check_timer();
    suspend_current_and_run_next();
}
......
```

再次运行，我们看到了这样的输出

```bash
>> hellostd
[ERROR] [kernel] trap_handler: Exception(IllegalInstruction) in application, bad addr = 0x464c457f, bad instruction = 0x0, kernel killed it.
Shell: Process 2 exited with code -4
```

这次的情况略有不同。出错的指令地址是 `0x0`，也就是说，用户程序尝试在 `0x0` 执行一条指令。显然，问题不在于 `0x0` 有什么，而在于用户程序是怎么跑到那里去的。我们用 `gdb` 来实时跟踪用户程序的执行过程。

## 精简 gdb 命令

一千个人有一千种方式使用 gdb，我们在此仅介绍最基本的一种。你可能会需要下面这些命令：

- `b *0x12345678` 表示在 `0x12345678` 这个地址设置断点
- `i` 可以查看各种信息。例如 `i b` 是查看断点，`i reg` 是查看所有通用寄存器
- `d 1` 可以删除 `1` 号断点。先用 `i b`查看断点编号再删除
- `x/10i $pc` 表示输出从 `pc` 当前位置开始往下的十条指令。`pc` 可以换成其他地址
- `si` 表示向下执行一条指令
- `n` 可以执行下一行代码，不进入本行执行的函数。对用户程序不一定有效
- `c` 表示继续运行，直到遇到下一个断点为止
- `ctrl+C` 可以打断运行，如果一次 `c` 执行太久没有相应可以试试这个
- `q` 可以退出 `gdb`

### 内核中运行 gdb 的注意事项

上面给出的命令中没有跟查看源代码或者函数符号相关的命令，这是因为 gdb 本质上是在对内核运行。而对于内核运行起来后实时加载的用户程序，gdb 无法掌握它们的符号信息，因而只能通过具体的地址进行调试，也只能看到汇编指令。

> 如果有 gdb 扩展或者其他项目支持查看内核中加载的用户程序的符号，可以联系我们，后续加进教程里

不过，查看内核的源代码是可以的。在 `os/Makefile` 下把第 2 行的 `MODE := release` 改为 `MODE := debug` 就可以了。

gdb 也不支持跨地址空间的查找。换句话说，它只知道当前能不能访问某个地址（虚拟地址），不会管现在的页表在哪，所以内核调试时经常会遇到因为地址当前无法访问而打不上断点的情况。这时可分为以下情况处理：

1. 把断点打在内核入口，也即 `0x80200000` 处，然后使用 `c` 命令跳过去。之后就可以打大部分内核符号的断点了。
2. 把断点打在 `mm::init()` （页表初始化函数）然后使用 `c` 命令跳过去，再用 `n` 指令跳过这段流程，就可以打页表中有映射的地址的断点了，例如跳板页 `TRAMPOLINE`。
3. 一般来说，如果想打用户程序的断点，可以把断点先打在内核的 `__alltraps` 和 `__restore` 上，这是在 `os/src/trap/trap.S` 中定义的从内核到用户程序的入口和异常中断的入口。然后手动执行直到它跳到用户程序的地址，之后就可以打用户地址的断点了。
   1. 但 `rCore-Tutorial` 实际上的 `__alltraps` 和 `__restore` 会被复制到跳板页 `TRAMPOLINE` 去执行。对于 `__alltraps`，它就是跳板页地址 `TRAMPOLINE`
   2. 对于 `__restore`，我们可以先 `b __restore` `b __alltraps` 查看这两个地址的偏移，然后算出它相对于 `TRAMPOLINE` 的偏移，从而在跳板页的对应位置打上断点

上面的过程实在比较麻烦，可以有以下几种办法改进

1. 先使用 `c` 命令，等待程序运行到 `user_shell` 等待输出的时候，再 `ctrl+C`，就可以打用户地址空间的断点了。但缺点是此时无法打内核的断点
2. 把上面断点的流程写进 gdb 脚本
3. 把初始化页表放在内核启动的最开头 `entry.asm` 里，可以省去第二步；使用单页表（见`rCore-Tutorial` 指导书 `ch4` 练习）把跳板页去掉，可以直接把断点打在 `__alltraps` 和 `__restore`，省去第三步。

## 使用 rCore-Tutorial 自带的命令调试

在 `os/Makefile` 的最下面有这些命令

```makefile
debug: build
	@tmux new-session -d \
		"qemu-system-riscv64 -machine virt -nographic -bios $(BOOTLOADER) -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA) -s -S" && \
		tmux split-window -h "riscv64-unknown-elf-gdb -ex 'file $(KERNEL_ELF)' -ex 'set arch riscv:rv64' -ex 'target remote localhost:1234'" && \
		tmux -2 attach-session -d


gdbserver: build
	@qemu-system-riscv64 -M 128m -machine virt -nographic -bios $(BOOTLOADER) -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA) \
	-drive file=$(FS_IMG),if=none,format=raw,id=x0 \
        -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 \
	-s -S

gdbclient:
	@riscv64-unknown-elf-gdb -ex 'file $(KERNEL_ELF)' -ex 'set arch riscv:rv64' -ex 'target remote localhost:1234'

```

下面尝试使用 `gdbserver` 和 `gdbclient` 进行调试。

首先打开两个终端，它们都切换到 `os/` 这个目录。然后在第一个终端执行 `make gdbserver`，在第二个终端执行 `make gdbclient`。如果在第二个终端看到下面的输出，表示连接成功

```
❯ make gdbclient
GNU gdb (SiFive GDB 9.1.0-2020.08.2) 9.1
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-linux-gnu --target=riscv64-unknown-elf".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://github.com/sifive/freedom-tools/issues>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
Reading symbols from target/riscv64gc-unknown-none-elf/release/os...
The target architecture is assumed to be riscv:rv64
Remote debugging using localhost:1234
warning: Architecture rejected target-supplied description
0x0000000000001000 in ?? ()
(gdb) 
```

先预想一下接下来发生什么事情：

1. `initproc` 先启动，然后是 `user_shell`
2. 我们会在 `user_shell` 里输入 `hellostd` 这个字符串
3. 之后 `os/src/task/process.rs:ProcessControlBlock::exec` 会加载 `hellostd` 这个测例。
4. 等到它运行到用户态时，会报错

由此可以设计出一种断点流程（`(gdb)`开头的内容都是在 `gdb client` 那个终端输入的）：

```
(gdb) b ProcessControlBlock::exec
(gdb) c
// 此时执行到该断点，程序暂停。看一下 gdb server 那个终端，得知是正在启动 user_shell
(gdb) c
// 此时没有执行到该断点。此时执行到该断点，发现终端已启动
// 切到 gdb server 那个终端输入
hellostd
// 此时执行到该断点，程序暂停。hellostd 测例准备启动，下一个断点打到 __restore
(gdb) b *0xfffffffffffff060
(gdb) c
// 此时执行到该断点，程序暂停。删除 __restore 的断点，因为它在进入用户态后会无法访问
(gdb) d 2
(gdb) si
// 按住回车不动，等 gdb 一直往下执行，直到切到用户地址
// 然后对一下反汇编结果，确认当前是否在 hellostd 测例中
(gdb) x/10i $pc
(gdb) si
// 按住回车不动，等 gdb 一直往下执行，直到切到用户地址，看哪里跳转到0
```

执行上面的流程，最后会发现是在 `0x608` 这个地址跳转到 `0` 的。看一下反汇编这个地址附近的指令：

```asm6502
     5fc:	a0878793          	addi	a5,a5,-1528
     600:	639c                	ld	a5,0(a5)
     602:	85b2                	mv	a1,a2
     604:	21010113          	addi	sp,sp,528
     608:	8782                	jr	a5
```

`0x608` 的一条 `jr a5` 跳转到了 `0x0`。这个错误的 `a5` 是在上面 `0x600` 的一条 `ld	a5,0(a5)` 加载的。

这时可以按 `q` 退出 gdb，重新走一遍上面的流程，在最后一步进入 `hellostd` 的用户态时，加一个断点 `b *0x600`，就可以 `c` 跳到这一条加载指令之前。这时通过 `i reg` 查看通用寄存器的值，可以得知 `a5` 的值是 `0x7000`。

那么 `0x7000` 的位置应该有值吗？如果它不是 `0`，又应该是多少呢？

## 收尾工作

内核里有一个函数会直接操作用户地址空间，就是 `os/src/mm/memory_set.rs:MemorySet::from_elf()`。自然地我们会想要加一条调试输出检查它加载 ELF 文件时每一段的地址：

```rust
pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
    let mut memory_set = Self::new_bare();
    // map trampoline
    memory_set.map_trampoline();
    // map program headers of elf, with U flag
    let elf = xmas_elf::ElfFile::new(elf_data).unwrap();
    let elf_header = elf.header;
    let magic = elf_header.pt1.magic;
    assert_eq!(magic, [0x7f, 0x45, 0x4c, 0x46], "invalid elf!");
    let ph_count = elf_header.pt2.ph_count();
    let mut max_end_vpn = VirtPageNum(0);
    for i in 0..ph_count {
        let ph = elf.program_header(i).unwrap();
        if ph.get_type().unwrap() == xmas_elf::program::Type::Load {
            let start_va: VirtAddr = (ph.virtual_addr() as usize).into();
            let end_va: VirtAddr = ((ph.virtual_addr() + ph.mem_size()) as usize).into();
            error!("start_va {:x} end_va {:x}", start_va.0, end_va.0);
    ......
```

再次运行测例，得到输出：

```bash
>> hellostd
[ERROR] start_va 0 end_va 5aac
[ERROR] start_va 6e70 end_va 7820
```

这个结果和其他测例有什么不同吗？可以再试试 `hello` `42` 乃至原本的 Rust 测例比如 `ch2b_power_3`，发现只有 `hellostd` 的 `start_va` 不是页对齐的。

继续看 `from_elf` 这个函数，会发现在下面这一句

```rust
memory_set.push(
    map_area,
    Some(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize]),
);
```

实际上是默认了每一个 `LOAD` 段都是页对齐的。因为它把来自 ELF 文件的段数据 `elf.input` 直接塞进 `map_area` 的区间里，没有做任何偏移。我们终于找到了 bug 的源头。于是我们将这段代码改成考虑偏移的版本：

```rust
if start_va.page_offset() == 0 {
    memory_set.push(
        map_area,
        Some(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize]),
    );
} else {
    let data_len = start_va.page_offset() + ph.file_size() as usize;
    let mut data: Vec<u8> = Vec::with_capacity(data_len);
    data.resize(data_len, 0);
    data[start_va.page_offset()..].copy_from_slice(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize]);
    memory_set.push(
        map_area,
        Some(data.as_slice()),
    );
}
```

其中 `else` 分支创建了一个 `data` `Vector`，它在页偏移的部分填 0，后面再存来自 `elf.input` 的真正的数据。

至此，我们终于解决了 `Shell: Process 2 exited with code -4` 的问题。