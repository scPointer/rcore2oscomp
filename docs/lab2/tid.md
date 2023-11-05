# 规范：TID 定义问题

`rCore-Tutorial` 的接口设计受到 `zircon` 影响，且为了教学简化了许多模块的设计。但在支持实际的 `Linux` 应用时，我们需要改掉这些设计。

本节介绍`rCore-Tutorial` 的 `tid` 与标准 `tid` 有何不同，之后会有更多这样的小节说明`rCore-Tutorial`其他模块中的规范问题。

### POSIX syscall 的定义

在 `Linux` 中，每个线程有**全局唯一**的线程ID，也就是 `tid`，且**每个进程的初始线程的 `tid` 等同于进程的 `pid`**。举个例子来说，可能是这样：

| 线程  | pid | tid |
| --- | --- | --- |
| a   | 1   | 1   |
| b   | 2   | 2   |
| c   | 1   | 3   |
| d   | 1   | 4   |
| e   | 5   | 5   |

这里 `a,c,d` 是同一个进程里的线程，`b` `e` 分别是独立的进程。再举几个例子：

- 如果往 `a,c,d` 里再加一个线程 `f`，它可能是 `pid=1,tid=6`，当然其他大于 6 的 `tid` 也是合法的。

- 如果进程 `d` 退出，然后其他进程再创建一个进程 `d2`，那么它可能是 `pid=4,tid=4`，也可能是其他的 `pid=x,tid=x(x>=6)`。总之，`tid`不能重复。

### rCore-Tutorial

而相对的，在 `rCore-Tutorial` 中，每个进程内部的每个线程的ID独立排序，且每个进程的初始线程的 `tid` 为 0。也就是像下面这样

| 线程  | pid | tid |
| --- | --- | --- |
| a   | 1   | 0   |
| b   | 2   | 0   |
| c   | 2   | 1   |
| d   | 3   | 0   |
| e   | 2   | 2   |
| f   | 1   | 1   |

这里 `a,f` 是同一个进程内的两个线程，`b,c,e` 是同一个进程内的三个线程，`d` 是一个独立进程。

### 实现建议

> 在通过比赛测例时，你需要修改 `tid` 实现以适应实际 Linux 应用的需求。但在本次实验中**不需要**改下面的实现。

所以我们在修改 `tid` 的实现时，可以把 `tid` 改成原来那个全局唯一的 `pid`，也即

- 每次生成一个进程时，为它分配一个全局ID作为 `tid`。然后令它的 `pid` 的值为 `tid`。

- 每次生成一个线程时，也为它分配一个全局ID作为 `tid`。然后令它的 `pid` 的值为生成它的线程的 `pid`（这表示它们俩在同一个进程里）

### 扩展阅读：waittid

如果认真看 `syscall`列表和 [`sys_wait` 的文档](https://man7.org/linux/man-pages/man2/waitpid.2.html)，你可能会注意到这里没有 `waittid`，只有 `wait,waitpid,waitid`（注意 `waitid=wait+id`）。那么 `Linux` 应用如何实现 `rCore-Tutorial` 的 `waittid` 呢？你可能已经在其他项目中见过它的形式了：

```c
int pthread_join(pthread_t, void **);
```

在 `musl` 源码中，`pthread_join` 最终会调用 `src/thread/pthread_join.c/__pthread_timedjoin_np`。我们可以简单分析这个函数是如何实现等待一次线程退出的：

```c
static int __pthread_timedjoin_np(pthread_t t, void **res, const struct timespec *at)
{
	int state, cs, r = 0;
	__pthread_testcancel();
	__pthread_setcancelstate(PTHREAD_CANCEL_DISABLE, &cs);
	if (cs == PTHREAD_CANCEL_ENABLE) __pthread_setcancelstate(cs, 0);
	while ((state = t->detach_state) && r != ETIMEDOUT && r != EINVAL) {
		if (state >= DT_DETACHED) a_crash();
		r = __timedwait_cp(&t->detach_state, state, CLOCK_REALTIME, at, 1);
	}
	__pthread_setcancelstate(cs, 0);
	if (r == ETIMEDOUT || r == EINVAL) return r;
	__tl_sync(t);
	if (res) *res = t->result;
	if (t->map_base) __munmap(t->map_base, t->map_size);
	return 0;
}
```

其中，

- 两次 `__pthread_setcancelstate` 分别表示关闭/打开取消请求。线程可以通过 `pthread_cancel` 函数去试图终止其他线程的运行，但其他线程也可以设置是否响应。在 `__pthread_timedjoin_np` 里的这两句 `__pthread_setcancelstate` 中间的代码运行时，当前线程是不会响应取消请求的。你可以类比内核中“关闭异常中断”的过程。
- `__timedwait_cp` 会使用一个 `sys_futex` 在某个内存地址设置一个条件变量。你可以将它理解为类似 `rCore-Tutorial ch8` 中条件变量一节的实现。
- 当当前线程所等待的那个线程退出时，它会修改表示线程状态的条件变量的值。这些条件变量的地址和上一节中我们“跳过实现”的 `sys_set_tid_address` 的参数有关。
- 这会唤醒正在等待的当前线程，使它获取返回结果 `r` 并继续执行。
