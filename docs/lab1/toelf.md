## 编译到二进制文件

在 `ch6` 中，`rCore-Tutorial` 指导书详细介绍了应用程序运行时如何读写文件、文件系统 `easy-fs` 的实现以及内核如何与文件系统交互。但对于用户程序如何在编译后被塞进文件系统这个过程，指导书讲得比较粗略，容易让大家以为用户程序和文件系统是以某种“魔法”联系在一起的。

现在我们先暂时跳出内核，破解这一层魔法。

### 分析 user/Makefile

在 `user `文件夹下，运行

```
make build CHAPTER=8 BASE=2
```

这一步会将所有 `ch8` 及之前的测例编译后放一份到 `user/build/elf/` 中，然后将每个应用删除 ELF header 符号得到纯二进制的镜像，放到 `user/build/bin/`，如同 `ch2` “实现应用程序”一节介绍的那样。但和 `ch2` 不同的是，这些ELF文件会被放到文件系统里，而不是直接链接进内核。

我们可以看 `user/Makefile` 这个文件了解 `make build` 时发生了什么：

```makefile
ELFS := $(patsubst $(APP_DIR)/%.rs, $(TARGET_DIR)/%, $(APPS))

binary:
	@echo $(ELFS)
	@if [ ${CHAPTER} -gt 3 ]; then \
		cargo build $(MODE_ARG) ;\
	else \
		CHAPTER=$(CHAPTER) python3 build.py ;\
	fi
	@$(foreach elf, $(ELFS), \
		$(OBJCOPY) $(elf) --strip-all -O binary $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.bin, $(elf)); \
		cp $(elf) $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.elf, $(elf));)

......

pre:
	@mkdir -p $(BUILD_DIR)/bin/
	@mkdir -p $(BUILD_DIR)/elf/
	@mkdir -p $(BUILD_DIR)/app/
	@mkdir -p $(BUILD_DIR)/asm/
	@$(foreach t, $(APPS), cp $(t) $(BUILD_DIR)/app/;)

build: clean pre binary
	@$(foreach t, $(ELFS), cp $(t).bin $(BUILD_DIR)/bin/;)
	@$(foreach t, $(ELFS), cp $(t).elf $(BUILD_DIR)/elf/;)

clean:
	@cargo clean
	@rm -rf $(BUILD_DIR)

```

`build: clean pre binary` 告诉我们 `build` 这个任务依赖于后面的三个任务，所以只需要把这几个任务单独拉出来看：

- `ELFS` 定义了所有测例对应的 ELF 文件。`patsubst` 这个命令的意思是，把 `$(APPS)` 中每一个形如 `$(APP_DIR)/%.rs` 这样的字符串找出来，把它替换成 `$(TARGET_DIR)/%`（符号 `%` 对应前面 `$(APP_DIR)/%.rs` 那里的 `%` 表示的字符串），然后把所有替换后的串交给 `ELFS`。比如 `src/bin/ch6_file0.rs` 会被替换成 `target/riscv64gc-unknown-none-elf/release/ch6_file0`。
- `clean` 删除了 `cargo` 也就是 Rust 编译后的临时文件，然后删除 `build` 目录
- `pre` 创建了 `build` 目录以及下属的几个子目录，`app` `elf` `bin` 分别存放测例代码、测例编译后的 ELF 文件、ELF 文件删除元信息后的纯可执行文件。然后把代码复制一份，存到 `app` 目录下。
- - `asm` 下是空的，它会在另一个叫 `disasm` 的任务被用到
- `binary` 首先用 `if [ ${CHAPTER} -gt 3 ];` 检查编译的章节是否大于 3，大于 3 则使用 Rust 的编译器 `cargo` 编译，否则使用 python。随后，它会把 `$(ELFS)` 中所有文件的调试段删掉，命名为对饮的 `.bin` 文件。然后把 `$(ELFS)` 中所有文件复制一份，命名为对应的 `.elf` 文件。
- - 如果你还记得的话，在 `ch3` 及之前我们的内核是“批处理操作系统”，所以需要依赖其他程序（代码框架里用的 python）做链接
- 最后 `build` 任务会把上一步 `binary` 生成的 `.bin` 和 `.elf` 文件分别复制一份到 `$(BUILD_DIR)/bin/` 目录和 `$(BUILD_DIR)/elf/` 目录。

### 检查二进制文件的内容

这一部分简要介绍编译生成的二进制文件的内容，更详细的信息可以参照 `rCore-Tutorial` 指导书的附录 B。

在 `user/build/elf/` 下运行 `rust-objdump --arch-name=riscv64 -ld ch6_file0.elf > debug.S`，可以在文件中看到像这样的输出

```asm6502
ch6_file0.elf:    file format elf64-littleriscv

Disassembly of section .text:

0000000000000000 <_start>:
; _start():
       0: 0d 71            addi    sp, sp, -352
       2: 86 ee            sd    ra, 344(sp)
       4: 2e fc            sd    a1, 56(sp)
       6: aa e0            sd    a0, 64(sp)
       8: aa f5            sd    a0, 232(sp)
       a: ae f9            sd    a1, 240(sp)
       c: 97 00 00 00      auipc    ra, 0
      10: e7 80 20 5f      jalr    1522(ra)
......
```

这是一个 elf 格式的可执行文件。如果你看过 rCore-Tutorial 指导书的附录 B 的话可能会对这个形式有印象。这个测例的可执行文件与整个 OS 的可执行文件并没有什么不同。例如你可以在文件中找到：

- `_start` 是整个程序的入口，也就是 `os/src/task/process.rs` 中调用 `MemorySet::from_elf` 拿到的 entry_point。
- 文件中的 `ecall` 指令实际上就是内核中看到的 `syscall` 调用的来源。在 `ecall` 指令之前通常在操作 `a7` 以及 `a0` `a1` `a2` 等寄存器，这就是内核在 `os/src/trap/mod.rs:trap_handler` 中看到的 `let result = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12], cx.x[13]]);` 所调用的几个寄存器。

下一节我们会详细带大家分析一个类似文件的内容。

虽然我们用 Rust 编译了它，并且用 `rust-objdump `区查看它的反汇编，但到这一步这个文件已经和 Rust 无关了。我们可以尝试改用前面安装的交叉编译工具链，运行 `riscv64-linux-musl-objdump -ld ch6_file0.elf > debug.S`，可以在文件中看到这样的输出

```asm6502
ch6_file0.elf:     file format elf64-littleriscv


Disassembly of section .text:

0000000000000000 <_start>:
_start():
       0:    710d                    addi    sp,sp,-352
       2:    ee86                    sd    ra,344(sp)
       4:    fc2e                    sd    a1,56(sp)
       6:    e0aa                    sd    a0,64(sp)
       8:    f5aa                    sd    a0,232(sp)
       a:    f9ae                    sd    a1,240(sp)
       c:    00000097              auipc    ra,0x0
      10:    5f2080e7              jalr    1522(ra) # 5fe <_ZN8user_lib9clear_bss17hb6a21b7ca700f785E>
......
```

虽然排版上略有不同，且对函数调用地址给出了注释信息，但二进制文件本身是相同的。这意味着，我们可以用交叉编译工具链中的 `gcc` 编译一个 C 程序，把它塞到文件系统镜像里，它应该也能像 `user` 库里的 Rust 测例一样正常运行。不过这里还有一些细节问题，我们之后再讲这件事。 
