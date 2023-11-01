### 编译到二进制代码

在 `ch6` 中，`rCore-Tutorial` 指导书详细介绍了应用程序运行时如何读写文件、文件系统 `easy-fs` 的实现以及内核如何与文件系统交互。但对于用户程序如何在编译后被塞进文件系统这个过程，指导书讲得比较粗略，容易让大家以为用户程序和文件系统是以某种“魔法”联系在一起的。

现在我们先暂时跳出内核，破解这一层魔法。

在 `user `文件夹下，运行

```
make build CHAPTER=8 BASE=2
```

这一步会将所有 `ch8` 及之前的测例编译后放一份到 `user/build/elf/` 中，然后将每个应用删除 ELF header 符号得到纯二进制的镜像，放到 `user/build/bin/`，如同 `ch2` “实现应用程序”一节介绍的那样。但和 `ch2` 不同的是，这些ELF文件会被放到文件系统里，而不是直接链接进内核。

接下来在 `user/build/elf/` 下运行 `rust-objdump --arch-name=riscv64 -ld ch6_file0.elf > debug.S`，可以在文件中看到像这样的输出

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

这是一个 elf 格式的可执行文件。如果你看过 rCore-Tutorial 指导书的附录 B，就应该对这个形式有印象。这个测例的可执行文件与整个 OS 的可执行文件并没有什么不同。虽然我们用 Rust 编译了它，并且用 `rust-objdump `区查看它的反汇编，但到这一步这个文件已经和 Rust 无关了。我们可以尝试改用前面安装的交叉编译工具链，运行 `riscv64-linux-musl-objdump -ld ch6_file0.elf > debug.S`，可以在文件中看到这样的输出

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
