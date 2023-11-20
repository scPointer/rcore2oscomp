# 编写独立的模块

在本节中，我们将展示如何实现简单的虚拟文件系统（`VFS`）和内存文件系统（`RamFs`）模块，并改造原有的`easy-fs` 以满足 `VFS` 的要求。

这需要使用 `rCore-Tutorial` 实验的仓库，但**不依赖于**你在 `rCore-Tutorial` 中完成的任何作业代码。你可以跟着这个例子做，也可以直接看[指导书演示中新增和修改的代码](https://github.com/Godones/rCore-Tutorial-v3/tree/ch8-vfs)。

## 虚拟文件系统 `VFS`

我们可以用 `cargo new` 命令生成一个新的 `Rust` 项目，并用 `--lib` 指定它是一个会被其他程序调用的库（`lib`），这样这个模块就不会包含 `main` 函数了。

```bash
cargo new libvfs --lib
```

在 `Rust` 中这样一个项目被称为 `Crate`。接下来打开这个新的 `Crate`，删除其中自带的初始代码，并在`lib.rs`的开头加上

```
#![cfg_attr(not(test), no_std)]
```

这一行对于大部分独立的**内核**模块是必要。这样在测试环境下，模块使用标准库 `std`；而在非测试模式下，模块不使用 `std`，此时我们的库是一个满足内核要求的 `no_std` 库。

> “测试环境”是由 `Rust` 编译器定义的。当运行 `cargo test` 时，会自动加上 `test` 属性，从而运行所有带有 `#[test]` 标记的代码
> 
> `cfg_attr(not(test), no_std)` 是一种宏，你可以类似理解为 C 语言的条件编译 `#ifdef...#define...`，它的意思是如果属性中包含 `not(test)`，也即不包含 `test`，那么给后续的代码加上 `no_std` 属性，也即不使用标准库。
> 
> 开头的 `#!` 表示它对整个 `Crate` 都生效。与之相对的，如果不加感叹号 `!` 则表示这一行条件编译只对接下来的一行或者一个代码块生效

在 `rCore-Tutorial ch7`，我们已经对 `easy-fs` 和文件系统的分层架构有了一个了解，并给出了一个 `Inode` 的抽象，这个抽象代表了磁盘上的一个文件或者目录，公开了几个重要的接口:

```rust
/// Inode struct in memory
pub struct Inode{...};
impl Inode{
     pub fn new(
        block_id: u32,
        block_offset: usize,
        fs: Arc<Mutex<EasyFileSystem>>,
        block_device: Arc<dyn BlockDevice>,
    ) -> Self;
    /// find the disk inode of the file with 'name'
    pub fn find(&self, name: &str) -> Option<Arc<Inode>>;
    /// create a file with 'name' in the root directory
    pub fn create(&self, name: &str) -> Option<Arc<Inode>> ;
    /// create a directory with 'name' in the root directory
    ///
    /// list the file names in the root directory
    pub fn ls(&self) -> Vec<String>;
    /// Read the content in offset position of the file into 'buf'
    pub fn read_at(&self, offset: usize, buf: &mut [u8]) -> usize;
    /// Write the content in 'buf' into offset position of the file
    pub fn write_at(&self, offset: usize, buf: &[u8]) -> usize;
    /// Set the file(disk inode) length to zero, delloc all data blocks of the file.
    pub fn clear(&self)
}
```

在内核中，`rCore-Tutorial` 把 `Inode` 进一步封装成 `OSInode` 。不过，内核中的 `OSInode` 必须和上述 `Inode` 一一对应吗？如果我们实现了另外一个文件系统，它也定义了自己的 `Inode` 结构并实现了上面这些接口，内核能不能同时使用这两个文件系统呢？

最靠没打满粗暴的方法是在 `OSInode` 增加成员函数和接口:

```rust
pub struct OSInode {
    ....
    inner: UPSafeCell<OSInodeInner>,
}
/// inner of inode in memory
pub struct OSInodeInner {
    offset: usize,
    inode: Arc<InodeType>,
}

pub enum InodeType{
    Easyfs(easy_fs::Inode),
    Otherfs(otherfs::Inode),
}

impl OSInode{
    pub fn new(readable: bool, writable: bool, inode: Arc<InodeType>) -> Self
}
```

但这样一来，在`OSInode`的实现中，每次进行读写或其它操作时，我们都需要使用一个 `match` 语句来区分每一个文件系统。这既影响效率的同时，代码也不是那么优雅。

Linux 的解决方法是使用一种虚拟文件系统（`VFS`）的框架，掌管所有的文件系统，规定了逻辑上目录树结构的通用格式及相关操作的抽象接口。不管是什么文件系统，只要实现了虚拟文件系统要求的那些抽象接口，就可以通过挂载 (`mount`) 等方式接入内核。这样一来，内核就可以用一个统一的逻辑目录树结构管理所有这些持久存储设备上的不同文件系统。

在 Linux 的 `VFS` 中，有几个很重要的数据结构和接口定义。这里我们只介绍一小部分，更多的定义可以在[`VFS` 文档](https://docs.kernel.org/filesystems/vfs.html) 查阅。

```c
struct super_operations {
        struct inode *(*alloc_inode)(struct super_block *sb);
        void (*destroy_inode)(struct inode *);
        void (*free_inode)(struct inode *);
        void (*dirty_inode) (struct inode *, int flags);
        int (*write_inode) (struct inode *, struct writeback_control *wbc);
        int (*drop_inode) (struct inode *);
        void (*evict_inode) (struct inode *);
        void (*put_super) (struct super_block *);
        ......
}
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        ......
}
struct inode_operations {
        int (*create) (struct mnt_idmap *, struct inode *,struct dentry *, umode_t, bool);
        struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
        int (*link) (struct dentry *,struct inode *,struct dentry *);
        int (*unlink) (struct inode *,struct dentry *);
        int (*symlink) (struct mnt_idmap *, struct inode *,struct dentry *,const char *);
        int (*mkdir) (struct mnt_idmap *, struct inode *,struct dentry *,umode_t);
        int (*rmdir) (struct inode *,struct dentry *);
        int (*mknod) (struct mnt_idmap *, struct inode *,struct dentry *,umode_t,dev_t);
        int (*rename) (struct mnt_idmap *, struct inode *, struct dentry *,
                       struct inode *, struct dentry *, unsigned int);
        ......
}
```

这里给出了超级块、文件、`Inode` 的接口定义。其中 `file_operations` 类似 `rCore-Tutorial` 的 `trait File`， `inode_operations` 类似 `easy-fs` 的 `Inode` 接口。

> 事实上， `easy-fs` 参考了许多 `Linus VFS` 的设计。
> 
> 不过在参考Linux的实现时，切勿直接仿照，以笔者的经验来看，直接将 Linux的 C语言实现翻译到 `Rust` 是不好的，这两门语言有自己的特点。在 C 语言中看起来很合理的东西（如指针）放到 `Rust` 中就不那么容易一一对应了，我们应该在参考 Linux 的设计思路前提下，设计一个利用 `Rust` 语言特性且符合 `Rust` 规范的结构。

前面我们提到，在实现其它类型的文件系统时，它们也可以实现一样的 `Inode` 接口。这样内核就可以方便地复用不同的文件系统。所以我们可以将这些接口组织成 `Rust` 的 `trait`，并放在刚才创建的 `libvfs` 中。现在这些接口本身就形成了一个独立可复用的模块，因为它不包含任何依赖。

```rust
/// The VfsInode trait
pub trait VfsInode:Send+Sync {
    /// find the disk inode of the file with 'name'
    fn find(&self, name: &str) -> Option<Arc<dyn VfsInode>>;
    /// create a file with 'name' in the directory
    fn create(&self, name: &str) -> Option<Arc<dyn VfsInode>>;
    /// list the file names in the root directory
    fn ls(&self) -> Vec<String>;
    /// Read the content in offset position of the file into 'buf'
    fn read_at(&self, offset: usize, buf: &mut [u8]) -> usize;
    /// Write the content in 'buf' into offset position of the file
    fn write_at(&self, offset: usize, buf: &[u8]) -> usize;
    /// Set the file(disk inode) length to zero, delloc all data blocks of the file.
    fn clear(&self);
}
```

你可能已经看到了这些接口出现了一些变化，之前的 `Arc<Inode>` 变成了 `Arc<dyn VfsInode>`, 去掉了对具体数据结构的依赖，在你新实现的文件系统中，你没有必要再定义一个跟`easy-fs` 一样的`Inode`的数据结构，你可以定义 `RamFsInode` 、 `MyInode` 等等，只要它实现了通用的 `trait` 就行。

我们没有对这些接口做太大的变更，也没有增加诸如 `mkdir` 、 `unlink` 这些想要兼容更多Linux应用就必须实现的接口，这是为了简化本实验的实现，避免对已有的`easy-fs`做出大量修改。对于初赛来说，目前已有的接口是够用的。反之，在真正开始设计你自己的 `VFS` 或者扩展往届内核实现的 `VFS` 时，就可能需要仔细地设计和添加新的接口。

在有了上面的`VfsInode` 定义后，我们就可以对 `easy-fs` 进行改造并实现我们定义的接口。首先在 `easy-fs`  的 `Cargo.toml`中，添加`libvfs` 依赖:

```rust
[dependencies]
libvfs = { path = "../libvfs" }
```

在原来的 `vfs.rs` 中，我们只需要将原来的实现移入到 `VfsInode` 的实现当中即可：

```rust
impl VfsInode for Inode {
    fn find(&self, name: &str) -> Option<Arc<dyn VfsInode>>{
        ......
    }
}
```

上面这些代码都很基础。因为我们只是在搭建一个 `VFS` 的框架，没有添加很多实际功能。但是当模块因为扩展逐渐变得复杂，我们仍然会需要添加大量的代码以满足应用需求。

为了能更好地利用好我们定义的`VfsInode` 接口，也为了说明这个独立模块确实有用，我们将根据这个接口实现一个内存文件系统（`RamFs`），也即完全不使用外部设备，把所有文件存在内存的文件系统。在 Linux 上通过 `ls /`查看根目录时，可以注意下面这些目录:

```
boot
dev
home
proc
sys
tmp
......
```

而其中 `tmp` 目录上挂载的临时文件系统（`TmpFs`）就类似内存文件系统。它除了使用系统内存外，还可以使用 `swap` 区域。我们还可以另外创建完全在内存中的 `RamFs` ，也可以挂载到 Linux 上。其它目录上其实还挂载了其它文件系统，比如 `sys` 对应 ` sysfs` 、`proc` 对应 `procfs` 等。广义上来说，这些文件系统其实也算是内存文件系统的一种，因为这些文件系统的信息是在系统运行过程中动态产生的，而这些信息位于内存中。

下面主要介绍如何从头开始实现一个简单的 `RamFs`。

## 内存文件系统 `RamFs`

类似 `libvfs`，再次新建一个`Crate`:

```rust
cargo new ramfs --lib
```

然后在这个新模块的 `Cargo.toml` 中加入对 `libvfs` 依赖，因为我们将要在这里实现 `libvfs` 定义的接口。

接下来我们定义 `RamFs` 的结构，就像 `EasyFileSystem` 一样:

```rust
pub struct RamFs<T> {
    inode_index: AtomicUsize,
    inode_count: AtomicUsize,
    root: Mutex<Option<Arc<RamFsDirInode<T>>>>,
    _provider: PhantomData<T>,
}
```

这里多出了一个泛型参数 `<T>`，后面再解释它的用法。`RamFs` 完全位于内存中，所以它的结构可以比磁盘文件系统简单许多。这里我们只使用了几个必须的变量来记录文件系统的一些元数据

1. `inode_index`：用来创建文件时分配inode号码
2. `inode_count`： 记录当前的文件数量，在目前的文件系统实现中其实和inode_index一样，因为我们没有提供删除文件的接口
3. `root`：记录当前的根目录
4. `_provider`: 是一个幽灵成员，“假装”这个类包含一个 `T` 类型的变量，以此绕过编译器的检查。你可以在[这篇专栏](https://zhuanlan.zhihu.com/p/533695108)中获取更多关于 `Rust` 的幽灵成员变量定义的细节。

接下来在 `RamFs` 中实现与 `EasyFileSystem` 类似的接口:

```rust
pub fn new() -> Arc<Self> {
        let fs = Arc::new(Self {
            inode_index: AtomicUsize::new(0),
            inode_count: AtomicUsize::new(0),
            root: Mutex::new(None),
            _provider: PhantomData,
        });
        fs
    }
pub fn root(self: &Arc<Self>) -> Arc<RamFsDirInode<T>> {
    let mut root = self.root.lock();
    if root.is_none() {
        let inode = Arc::new(RamFsDirInode::new(0, String::from("/"), self.clone()));
        *root = Some(inode.clone());
        self.inode_count.fetch_add(1, Ordering::SeqCst);
        self.inode_index.fetch_add(1, Ordering::SeqCst);
    }
    root.clone().unwrap()
}
```

再然后，我们定义两个最重要的数据结构：

```rust
pub struct TimeSpec {
    pub sec: usize,
    pub nsec: usize,
}
pub struct RamFsDirInode<T> {
    id: usize,
    name: String,
    children: Mutex<Vec<Arc<RamFsFileInode>>>,
    fs: Arc<RamFs<T>>,
    ctime: TimeSpec,
}
pub struct RamFsFileInode {
    id: usize,
    name: String,
    content: Mutex<Vec<u8>>,
    ctime: TimeSpec,
}
```

与 `easy-fs` 不同，在 `RamFs` 中定义了两种类型的 `Inode`。这是因为在文件系统中除了普通文件外还有目录、设备文件、链接文件等。而在定义 `VfsInode` 时， 我们说这个 `trait` 抽象了“文件”的操作，这里的“文件”其实包含了前面提到的所有种类的文件。但实际上这些“文件”的属性和需要的接口是不一样，例如目录不需要 `write_at` 或者 `read_at`，而普通文件不需要 `create` 或者 `ls`。所以虽然 `VfsInode` 抽象了所有种类的文件的操作，但还是要根据具体的文件类型做区分。简单起见，我们只定义了目录和普通文件两种类型。

> 在 `easy-fs` 中对文件类型进行了进一步的简化，从而没有区分目录和实际保存数据的文件。如果用户程序错误读取了目录文件，例如直接对根目录调用`read_at` ，就可能会导致文件系统崩溃。

上面已经定义了对应不同文件的`Inode`结构，接下来只需要为这两个结构实现 `libvfs` 中定义的`trait`。当然，这个实现不是唯一的，你可以自行修改:

```rust
impl VfsInode for RamFsFileInode {
    fn find(&self, _name: &str) -> Option<Arc<dyn VfsInode>> {
        None
    }
    fn create(&self, _name: &str) -> Option<Arc<dyn VfsInode>> {
        None
    }
    fn ls(&self) -> Vec<String> {
        Vec::new()
    }
    fn read_at(&self, offset: usize, buf: &mut [u8]) -> usize {
        let content = self.content.lock();
        let len = content.len();
        let buf_len = buf.len();
        let copy_start = offset.min(len);
        let copy_end = (offset + buf_len).min(len);
        buf[..(copy_end - copy_start)].copy_from_slice(&content[copy_start..copy_end]);
        copy_end - copy_start
    }
    fn write_at(&self, offset: usize, buf: &[u8]) -> usize {
        let mut content = self.content.lock();
        let len = content.len();
        let buf_len = buf.len();
        if len < offset + buf_len {
            content.resize(offset + buf_len, 0);
        }
        let copy_end = offset + buf_len;
        content[offset..copy_end].copy_from_slice(buf);
        buf_len
    }
    fn clear(&self) {
        let mut content = self.content.lock();
        content.clear();
    }
}
```

```rust
impl<T: RamFsProvider> VfsInode for RamFsDirInode<T> {
    fn find(&self, name: &str) -> Option<Arc<dyn VfsInode>> {
        let children = self.children.lock();
        for child in children.iter() {
            if child.name() == name {
                return Some(child.clone());
            }
        }
        None
    }
    fn create(&self, name: &str) -> Option<Arc<dyn VfsInode>> {
        let mut children = self.children.lock();
        let inode = Arc::new(RamFsFileInode::new(
            self.fs.alloc_inode(),
            String::from(name),
            T::current_time(),
        ));
        children.push(inode.clone());
        Some(inode)
    }
    fn ls(&self) -> Vec<String> {
        let children = self.children.lock();
        let mut names = Vec::new();
        for child in children.iter() {
            names.push(child.name().to_string());
        }
        names
    }
    fn read_at(&self, _offset: usize, _buf: &mut [u8]) -> usize {0}
    fn write_at(&self, _offset: usize, _buf: &[u8]) -> usize {0}
    fn clear(&self) {}
}
```

可以看到，`RamFsFileInode` 没有实现`create` `ls` `find`, `RamFsDirInode` 没有实现 `read_at` `write_at` `clear`，这符合普通数据文件和目录的定义。

不过，上面给出的这一套接口设计并不是唯一的。另一种思路是，在调用不支持的功能时（例如对 `RamFsFileInode` 调用 `create` ）返回错误而不是上述实现中的空值 `None`。在这种情况下，返回值可以设计为`Result<Arc<dyn VfsInode>,VfsError>`, 这需要在 `libvfs` 中额外定义文件系统可能出现的错误 `VfsError`。

> 选择使用类似 `VfsError` 这样的结构可以给出更详细的报错信息，但也需要使用者花更多的时间去了解什么是 `VfsError`。
> 
> 使用 `None` 给出的信息是含糊的，但使用者不需要引入额外的类就可以大致了解它的意思是这个操作“不支持”或者“出错了”。
> 
> 总之，使用自定义的结构体还是更通用的 `None` `Err`等等是编写模块时的一种选择，没有绝对的对错之分。

现在我们就已经根据 `VfsInode` 的定义实现了一个新的文件系统。在继续修改内核代码使得内核可以支持多种文件系统的功能前，我们先给出 `RamFs` 中泛型参数 `<T>` 的解释。

### `RamFs` 的泛型参数

一个文件系统在创建文件的时候要保存的哪些元数据？可以在 Linux 上使用 `stat` 命令查看一个文件，可以得到类似下面的信息：

```bash
stat ./Cargo.toml
  File: ./Cargo.toml
  Size: 272             Blocks: 8          IO Block: 4096   regular file
Device: 820h/2080d      Inode: 824376      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/ godones)   Gid: ( 1000/ godones)
Access: 2023-11-19 20:34:20.269529331 +0800
Modify: 2023-11-19 20:34:20.259529332 +0800
Change: 2023-11-19 20:34:20.259529332 +0800
 Birth: 2023-11-19 19:56:29.299676521 +0800
```

其中底部的四行显示了文件系统保存的关于文件的时间相关的元数据，包括创建时间、访问时间、修改时间等。在比赛中，确实也会涉及一些读取文件系统创建、修改时间的 `syscall`。所以当我们实现一个真正可用的文件系统时，需要它记录文件的时间信息。而文件系统只能通过操作系统获取时间，它本身没有办法直接从时钟设备中获取时间。所以这时候“获取时间”就变成了文件系统需要与内核进行交互的那部分功能。如果想要独立文件系统模块，而不是让文件系统依赖一个类似 `../crates/time` 的模块，就可以用泛型参数来解决。

关于 `rust` 的泛型参数，可以在网上搜到许多介绍的文档，这里不再赘述。直接来看这个泛型参数所使用的约束:

```rust
pub trait RamFsProvider: Send + Sync {
    fn current_time() -> TimeSpec;
}
```

我们定义了一个 `RamFsProvider` 来表达创建 `RamFs` 需要的外部输入，即获取当前的时间。有了这个约束，当我们在内核创建`RamFs`时，我们就必须为其提供对应的实现，而 `RamFs` 也就成功地从内核获得了额外的信息。

```rust
struct Provider;
impl ramfs::RamFsProvider for Provider {
    fn current_time() -> ramfs::TimeSpec {
        TimeSpec { sec: 0, nsec: 0 }
    }
}

fn main() {
    let ramfs = RamFs::<Provider>::new();
}
```

这个方法对于很多可能独立的模块很有用。通过 `trait` 和泛型参数，我们可以有效隔离模块和内核降低耦合。不过泛型方法也有自己的缺点：它具有传染性，导致我们很多更上层的数据结构都不得不带上泛型参数和约束。

当然也有一些不太安全的方式可以解决上面的问题，这里只作介绍，不建议大家使用。例如可以用 `rust` 的 `FFI` 将函数转换为 C 语言接口：

```rust
extern "C" {
    fn current_time() -> TimeSpec;
}
```

通过定义一个`extern "C"`，我们可以省去上面的那些泛型参数和相关定义，在需要获取时间时，直接就调用**全局可见**的符号 `current_time`。在 `Rust` 使用这个函数时，需要加上 `unsafe` 块，因为编译器不能确定这个函数的定义和数据所有权是否符合 `Rust` 规范。这其实就相当于声明了一个**全局可见**的函数，我们要在内核提供一个实现以便在编译链接时可以找到：

```rust
#[no_mangle]
fn current_time() -> TimeSpec{

}
```

这种方式的问题在于，定义的接口不能出现 `not FFI-safe` 的参数或返回值（当然强制出现也不是不可以，不过出现bug时可能比较难调试）。而且我们在调用时不得不出现 `unsafe` 代码。`#[no_mangle]` 的出现也可能导致实现冲突的情况，因为你不能保证你引入的库中没有定义一个相同的函数。

> `unsafe` 的直观解释是，这个函数或者这段代码不能满足 `Rust` 编译器提供的内存安全控制规范，因此需要程序员自己保证它符合规范。从某种程度上来说，它有点像安装软件时你按下的“同意此服务条款”。所以使用 `unsafe` 关键字时必须仔细检查内部的代码是否有内存安全问题。你可以在[官方文档的这一节](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html)找到更多的说明和讨论。

这里我们可以再提供一种方法: 直接参数传递。对于 `RamFs` 来说，可能的形式如下:

```rust
/// create a file with 'name' in the directory
fn create(&self, name: &str,ctime: TimeSpec) -> Option<Arc<dyn VfsInode>>;
```

这段代码在 `create` 函数中增加了一个 `ctime` 参数, 让用户可以在调用这个函数时将时间值传递进来。这种方式可能导致很多接口都不得不带上这个参数，降低效率，并抬高部分使用成本，而且不是在所有情况下适用。不过，它可以代替泛型实现，让内核代码更简洁。

## 修改内核以接入 `ramfs` 和 `VFS`

现在我们对内核进行修改，以便引入我们实现的 `libvfs` 和 `RamFs`。在 `os` 的 `Cargo.toml` 中插入

```rust
libvfs= {path = "../libvfs"}
ramfs = {path = "../ramfs"}
```

然后在内核的 `os/src/fs/inode.rs` 修改两个定义:

```rust
pub struct OSInodeInner {
    offset: usize,
    inode: Arc<dyn VfsInode>,
}

impl OSInode {
    /// create a new inode in memory
    pub fn new(readable: bool, writable: bool, inode: Arc<dyn VfsInode>) -> Self {
    }
}
```

为了正常运行，你可能还需要在 `easy-fs-fuse` 中引入依赖。因为这个程序使用了 `write_at` 功能，而这个功能现在被定义在`VfsInode` 中了。

这里还有一个前面没有提到的问题：我们在 `easy-fs` 或者 `RamFs` 中实现 `VfsInode` 时，我们应该在顶层模块中**导出**这个`trait`定义，例如

```rust
pub use libvfs::*;
```

因为对于使用这些模块的人来说，它可能并不想引入 `libvfs` 这个依赖, 我们尽量做到自包含来避免这样的问题，同时给予使用者最大的灵活性。

现在的 `rCore-Tutorial` 还用不到 `RamFs`，这是因为内核目前不支持挂载文件系统的功能。如果你基于 `rCore-Tutorial` 开发，会在比赛过程中用到它，当然也可以只看[上述修改的一个示例](https://github.com/Godones/rCore-Tutorial-v3/tree/ch8-vfs)，把本节学到的内容用在自己的内核中。只要文件系统实现了类似本节介绍的接口，就可以替换掉 `easy-fs`。

文件系统这部分的模块都是相对独立的。你可以使用现有的文件系统模块，别人也可以基于你的 `VFS` 定义去扩展和实现他们的文件系统，或者直接使用你实现的`easy-fs` 和 `RamFs` ，这几个模块都不依赖于内核，并且互相之间也不是强依赖，可以用简单的办法消除。

到现在为止，我们已经讲解了编写一个独立模块所需要的所有步骤。

## 可选实现：FAT32 格式的文件系统

到目前位置，我们已经接触了三个与文件系统有关的模块，分别是 `libvfs` `RamFs` `easy-fs`。以此为基础，可以实现比赛初赛所要求的 `FAT32` 格式的文件系统。我们在[第一章实验的介绍](../lab1/intro.md#在实验之后)提到过它。

`FAT32` 的格式比较复杂，但不需要从零写起，可以利用已有的开源项目，例如[`rust-fatfs`](https://github.com/rafalh/rust-fatfs)，然后在自己的 `rCore-Tutorial` 项目中支持它。也可以参考往届内核的实现，例如 [`Starry` 的 `axfs`](https://github.com/Azure-stars/Starry/tree/main/modules/axfs)。

这可以作为本章的作业，但你也可以选择把本章[操作系统与模块划分一节](./modular.md)和[对独立模块的要求一节](./module.md)中提到的分离出独立模块当作本章的作业。选择其中一项即可。