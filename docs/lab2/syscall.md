# 通过 Manual Page 添加 Syscall

加上上一章里对 `os/src/mm/memory_set.rs:MemorySet::from_elf()` 的修改，现在我们再次运行 `hellostd` 测例。

不出意外的话，输出应该是（根据你原仓库的实现，报错行号可能略有不同）：

```bash
>> hellostd
[kernel] Panicked at src/syscall/mod.rs:168 Unsupported syscall_id: 96
```

坏消息是，它仍然没有运行成功；好消息是，这实际上已经是最后一步了！只要我们按照 `rCore-Tutorial` 那样填补上缺失的 `syscall`，就可以成功运行这个测例。不过，这时候就不是指导书来定义这些 `syscall` 了，而是需要查看通用的 `syscall` 标准。

##### Syscall 定义

通常来说，`syscall` 定义可以在 `man7.org` 查到，比如[这是一个 `sys_clone` 的定义](https://man7.org/linux/man-pages/man2/clone.2.html)，里面包含了你可能需要知道的绝大部分东西，包括

- 接口定义和参数列表
```
int clone(int (*fn)(void *_Nullable), void *stack, int flags,
        void *_Nullable arg, ...  /* pid_t *_Nullable parent_tid,
                                    void *_Nullable tls,
                                    pid_t *_Nullable child_tid */ );
```
- syscall 的语义以及各个参数的语义。这包括关于 `syscall` 本身的描述和各个参数的描述。如果参数包括标志位，还会提每一个标志位的含义。不过这个页面并不会告诉你在哪一个位。下面节选一些 `sys_clone` 的定义

```
DESCRIPTION
       These system calls create a new ("child") process, in a manner
       similar to fork(2).

       By contrast with fork(2), these system calls provide more precise
       control over what pieces of execution context are shared between
       the calling process and the child process.
......
   The clone() wrapper function
       When the child process is created with the clone() wrapper
       function, it commences execution by calling the function pointed
       to by the argument fn.  (This differs from fork(2), where
       execution continues in the child from the point of the fork(2)
       call.)  The arg argument is passed as the argument of the
       function fn.
    clone3()
......
   Equivalence between clone() and clone3() arguments
......
   The flags mask
       CLONE_CHILD_CLEARTID (since Linux 2.5.49)
              Clear (zero) the child thread ID at the location pointed
              to by child_tid (clone()) or cl_args.child_tid (clone3())
              in child memory when the child exits, and do a wakeup on
              the futex at that address.  The address involved may be
              changed by the set_tid_address(2) system call.  This is
              used by threading libraries.

       CLONE_CHILD_SETTID (since Linux 2.5.49)
......
       CLONE_CLEAR_SIGHAND (since Linux 5.5)
......

```

- 返回值
- 错误类型。在 `rCore-Tutorial` 中，如果一个 `syscall` 执行失败，通常是直接返回 `-1` 的。但在 `POSIX syscall` 中，`syscall` 可以返回不同的负数，代表不同的错误类型。一些标准库程序会通过判断不同的错误类型来执行不同的代码，所以 debug 时别忘了你给用户程序返回的错误类型也可能出错。
- 其他需要说明的问题，比如存在的版本冲突、历史版本问题、额外的说明、代码示例等等
- 一些链接，可以点到相关的其他 `syscall`

##### 如何搜索到Syscall定义

一般来说，如果你知道 `syscall` 的名字。总是可以用搜索引擎找到上面的页面的。

但我们现在只知道 `syscall` 的 ID 是 `96`，怎么知道它是谁呢？

- 可以用网上整理好的博客，比如[这一篇](https://jborza.com/post/2021-05-11-riscv-linux-syscalls/)，在网页里搜索 `96` 就能找到我们想要的 `sys_set_tid_address`。
- 可以通过源代码查询。例如我们在[最初的实验](../lab1/intro.md#实验准备) 中下载的交叉编译工具链使用的 `musl-libc`。可以在[这里](https://git.musl-libc.org/cgit/musl) 下载到 `musl` 库的源代码，然后在目录下的 `arch/riscv64/bits/syscall.h.in` 中找到 `syscall` 对应的名字。

##### 回到主线

至少在参加比赛这个阶段，当我们要实现一个 `syscall` 时**不需要**实现定义里的所有内容。定义中 90% 的部分可能只会有不到 1% 的应用程序会使用。因此，我们只希望支持最基本的功能，让用户程序成功运行就行，更详细的定义可以以后需要时再加。

现在我们需要实现 `96` 号 `syscall` `set_tid_address`，你可以在[这里](https://man7.org/linux/man-pages/man2/set_tid_address.2.html)找到它的描述。

观察一下它的描述：

```
SYNOPSIS

       #include <sys/syscall.h>      /* Definition of SYS_* constants */
       #include <unistd.h>

       pid_t syscall(SYS_set_tid_address, int *tidptr);

       Note: glibc provides no wrapper for set_tid_address(),
       necessitating the use of syscall(2).

DESCRIPTION

       For each thread, the kernel maintains two attributes (addresses)
       called set_child_tid and clear_child_tid.  These two attributes
       contain the value NULL by default.

       set_child_tid
              If a thread is started using clone(2) with the
              CLONE_CHILD_SETTID flag, set_child_tid is set to the value
              passed in the ctid argument of that system call.

              When set_child_tid is set, the very first thing the new
              thread does is to write its thread ID at this address.

       clear_child_tid
              If a thread is started using clone(2) with the
              CLONE_CHILD_CLEARTID flag, clear_child_tid is set to the
              value passed in the ctid argument of that system call.

       The system call set_tid_address() sets the clear_child_tid value
       for the calling thread to tidptr.

       When a thread whose clear_child_tid is not NULL terminates, then,
       if the thread is sharing memory with other threads, then 0 is
       written at the address specified in clear_child_tid and the
       kernel performs the following operation:

           futex(clear_child_tid, FUTEX_WAKE, 1, NULL, NULL, 0);

       The effect of this operation is to wake a single thread that is
       performing a futex wait on the memory location.  Errors from the
       futex wake operation are ignored.
RETURN VALUE

       set_tid_address() always returns the caller's thread ID.

ERRORS

       set_tid_address() always succeeds.

```

会发现这个 `set_child_tid` 和 `clear_child_tid` 的描述都是用 `If a thread is started using clone(2) with...` 开头的，这里的 `clone` 是 `sys_clone`，本质上和我们的 `sys_fork` 同功能，但提供更多的参数。因为目前内核里对 `fork` 没有更多的参数支持，我们也显然不会满足这里提到的 `If`，所以这两个情况目前都是不必要的。

由于 `clear_child_tid` 在目前的内核中不存在，所以下一行说的 `sets the clear_child_tid value for the calling thread to tidptr.` 的情况也可以忽略，这样连这个 `syscall` 的唯一参数 `tidptr` 也可以不用管，我们直接看返回值就好了。

> 选择忽略哪些描述、实现哪些描述并不简单。一般情况下，可以运行一个用户程序，看它在调用这个 `syscall` 的时候给了哪些参数，然后在内核中选择性忽略它没有给的参数。不过有时候用户程序给出的参数也是不必要的，这需要反复 `debug` 才能知道。

返回值告诉我们，这里需要返回调用者的线程 ID。考虑 `rCore-Tutorial` 和实际 `Linux` 的线程定义差异（**下一节详细说明**），我们在此返回 `pid`。这样我们直接使用已有的 `sys_getpid()` 函数就好了。

**注意，返回 `sys_getpid` 而不是 `sys_gettid`！**

**重复一遍，返回 `sys_getpid` 而不是 `sys_gettid`！**

我们在 `os/src/syscall/mod.rs` 添加上这个 `syscall`：

```rust
/// set_tid_address syscall
pub const SYSCALL_SET_TID_ADDRESS: usize = 96;

/// handle syscall exception with `syscall_id` and other arguments
pub fn syscall(syscall_id: usize, args: [usize; 4]) -> isize {
    match syscall_id {
        ......
        SYSCALL_SET_TID_ADDRESS => sys_getpid(),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```

再次运行程序...随后它就会卡在另一个 `syscall` 上：`Unsupported syscall_id: 29`。

不过接下来就是你的任务了：看看布置实验那一节的要求，补上缺失的 `syscall` 使得 `hellostd` 测例运行成功。