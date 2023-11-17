# 引入外部代码

我们希望修改用户栈的实现，以满足标准库的需求。具体来说，用户栈需要满足上一章中提到的 [ELF 文件规范](https://articles.manugarg.com/aboutelfauxiliaryvectors.html)：

```
position            content                     size (bytes) + comment
  ------------------------------------------------------------------------
  stack pointer ->  [ argc = number of args ]     8
                    [ argv[0] (pointer) ]         8   (program name)
                    [ argv[1] (pointer) ]         8
                    [ argv[..] (pointer) ]        8 * x
                    [ argv[n - 1] (pointer) ]     8
                    [ argv[n] (pointer) ]         8   (= NULL)

                    [ envp[0] (pointer) ]         8
                    [ envp[1] (pointer) ]         8
                    [ envp[..] (pointer) ]        8
                    [ envp[term] (pointer) ]      8   (= NULL)

                    [ auxv[0] (Elf32_auxv_t) ]    16
                    [ auxv[1] (Elf32_auxv_t) ]    16
                    [ auxv[..] (Elf32_auxv_t) ]   16
                    [ auxv[term] (Elf32_auxv_t) ] 16  (= AT_NULL vector)

                    [ padding ]                   0 - 16

                    [ argument ASCIIZ strings ]   >= 0
                    [ environment ASCIIZ str. ]   >= 0

  (0xbffffffc)      [ end marker ]                8   (= NULL)

  (0xc0000000)      < bottom of stack >           0   (virtual)
  ------------------------------------------------------------------------
```

注意因为目标架构 `riscvgc64` 是 64 位，所以网页中的 4 在我们这里换成了 8，而 8 换成了 16。

你当然可以手动实现上面的内容，但我们想趁机向你介绍如何引入其他内核的实现来完成这个任务。

## 找到对应模块

一般来说，每一届操作系统比赛都会沿用往年的测例，然后添加新测例，因此这些通用的功能很可能就是往届内核里造过的“轮子”，而参考这些内核的实现将会是绝大部分同学不得不经历的过程。比赛本身**并不**排斥大家用往年的实现，只要在文档里说明来源即可。

> 那么这又引出另一个问题，参加比赛能不能直接沿用往届的完整内核来开发呢？当然可以！
> 
> 事实上，下一章的实验就是基于一个往届的内核来做的。不过，往年内核的功能不能算是你在本届比赛中的贡献，而别人的代码总是会比自己的代码难理解许多，所以单纯想一口吃成胖子只会让后续的开发举步维艰。

我们需要实现的模块在大部分内核中被叫做 `ELF Loader`，因为初始化用户栈、生成环境变量等信息发生在“用户程序加载进内核”的过程中。你可以在[比赛页面](https://os.educg.net/#/)查询往年内核实现赛的获奖名单，然后在 `github` 或者 [官方 gitlab](https://gitlab.eduxiji.net/explore) 上搜索对应的内核项目。在本次实验中，我们用[这个内核](https://github.com/scPointer/maturin)的 `kernel/src/loaders` 模块。

我们开发的是 Rust 内核，所以最好的情况当然是直接使用包装成 `Rust Crate` 的模块，这样我们只需要在 `os/Cargo.toml` 里引入模块作为依赖，就可以在内核中使用它了。不过就目前来说，操作系统比赛的内核实现中的大部分还没有这么高的模块化，只能使用复制粘贴代码的方式来“复用模块”。但模块化是本指导书的目标之一，在后续实验中我们会实现一个独立于内核的 `Rust Crate`，它支持某项特定的内核功能。

## 修改模块以适配内核

在测例仓库的 `lab2` 分支的 `loaders` 子目录下有**已经修改好**的模块，可以按照测例仓库 `README.md` 中提到的方法，把它**直接放进**你的 `os/src` 下。

本节剩下的内容只是在展示和解释我们修改原始模块的过程，以便你学习如何为内核添加其他模块。你当然可以用另一种方式精简这个模块或者使用其他模块。

### 允许新模块的 `warning` 以及缺失文档问题

首先，我们复制[这个文件夹](https://github.com/scPointer/maturin/blob/master/kernel/src/loaders)的内容到 `rCore-Tutorial` 的 `os/src` 目录下，然后在 `os/src/main.rs` 中引入这一模块（添加一条 `pub mod loaders`），尝试直接运行 `make run`。不出意外地它报错了，其中一类错误如下：

```bash
error: unused import: `sections::SectionData`
  --> src/loaders/mod.rs:19:5
   |
19 |     sections::SectionData,
   |     ^^^^^^^^^^^^^^^^^^^^^
   |
note: the lint level is defined here
  --> src/main.rs:22:9
   |
22 | #![deny(warnings)]
   |         ^^^^^^^^
   = note: `#[deny(unused_imports)]` implied by `#[deny(warnings)]`
```

报错信息指出，其中有一些变量没有被使用到。为什么这样的情况不是 `warning` 而是 `error` 呢？因为 `main.rs` 在开头特地指明了 `#![deny(missing_docs)]` 和 `#![deny(warnings)]`，表示所有缺失文档的文件、类、函数乃至所有 `warning` 都不被允许。这是一个非常好的特性，可以有效改善代码质量，但在调试时我们可以针对对应模块暂时取消它：

```rust
// at os/src/main.rs
#[allow(missing_docs)]
#[allow(warnings)]
pub mod loaders;
```

### 删掉不需要的功能和所有不存在的引用

看一下这个模块里的代码，发现它其实做了等同于 `rCore-Tutorial` 内核中 `os/src/mm/memory_set.rs:MemorySet::from_elf()` 的事情，也就是解析 ELF 文件的每一个 `LOAD` 段并塞入用户页表中。但我们只想要它处理用户栈的部分，也即：

```rust
        let info = InitInfo {
            ......
        };

        info!("info {:#?}", info);
        let init_stack = info.serialize(stack_top);
        debug!("init user proc: stack len {}", init_stack.len());
        stack_pma.write(USER_STACK_SIZE - init_stack.len(), &init_stack)?;
        stack_top -= init_stack.len();
```

这一部分，所以我们可以把 `init_vm` 函数里的其他部分删掉（除了获取 `elf_base_vaddr` 变量的一段，因为这个变量会被 `info = InitInfo {... }` 用到）。其他用不到的函数和类也可以删掉，只留下 `ElfLoader` 里的 `new` 和 `init_vm`。然后把每个文件开头用不到的引用以及不存在的模块引用删掉，它们并不属于我们的 `rCore-Tutorial` 内核。

### 替换原代码中的函数参数、返回值、常量

接下来就需要分门别类分析各个报错的位置了。举三个例子：
- `ElfLoader::new` 函数的返回值 `OSResult` 这个类不存在，但看函数实现可以发现它其实是想返回 `ElfLoader` 这个类本身或者返回一个报错字符串。所以我们直接将返回类型改为 `Result<Self, &str>`，错误消失了。
- `ElfLoader::init_vm` 的参数中，有一个 `vm: &mut MemorySet,`。`MemorySet` 看起来和我们内核中的 `MemorySet` 是类似的，但通过 `vm: &mut MemorySet` 输入会报错。回顾 `os/src/task/process.rs:ProcessControlBlock::exec` 函数，我们使用 `memory_set` 的方式应该是获取一个 `usize` 类型的 `memory_set.token()`，所以这里我们把这个参数替换成 `memory_token: usize`。此外，由于我们的  `os/src/mm/memory_set.rs:MemorySet::from_elf()` 已经生成了用户栈底地址，所以还要加一个参数 `stack_top: usize`
- 代码中用到一些常量，如 `PAGE_SIZE` `USER_STACK_SIZE` 可以用我们内核中 `os/src/config.rs` 中对应的常数代替。如果不清楚常数的含义，可以选择去查这个模块来源的内核。

总之，我们将替换原代码中的函数参数、返回值、常量，改成自己内核中对应的值

### 必要时增写函数或者功能

最后，我们还需要处理代码中用到但我们目前内核中没有的功能。通过查模块来源的内核的注释（或者通过上一节教的全局搜索）可以得知，原代码中的 `stack_pma.write(USER_STACK_SIZE - init_stack.len(), &init_stack)?;` 一行是将 `init_stack` 里的内容全部写到用户栈上。我们的内核没有这么方便的函数，但有一个类似的 `translated_byte_buffer`，可以利用它把原代码的这一行改写成：

```rust
let stack = translated_byte_buffer(memory_token, stack_top as *const u8, init_stack.len());
// 接下来要把 init_stack 复制到 stack 上
let mut pos = 0;
for page in stack {
    let len = page.len();
    page.copy_from_slice(&init_stack[pos..pos + len]);
    pos += len;
}
assert!(pos == init_stack.len());
```

之后，把模块前的 `#[allow(missing_docs)]` 和`#[allow(warnings)]` 删掉，就几乎得到在测例仓库的 `lab2` 分支的 `loaders` 子目录的模块了。

> 部分同学可能会有疑问：`info = InitInfo {... }` 里定义的 `auxv` 以及用到的 `elf_base_vaddr` 事实上也都是没用的，全删掉也不影响运行，为什么测例仓库给出的模块没有处理它们？
>
> 这是因为我们在添加一个新模块时，很难准确判断哪些东西是不必要的。在本实验中我们要用“用户栈的预处理”这一部分，其他的可以删除，但并不知道这一部分之中哪些是必要的。在这种情况下，尽可能保留原有模块的部分在某种程度上可以提高运行成功的概率。等到我们成功运行这个模块之后，可以再来考虑删掉哪些部分。
>
> 事实上，在比赛决赛第一阶段的 `libc-test` 测例中，需要使用到原 `loaders` 模块的所有内容，所以所有删掉的代码最后都是要加回来的。