[![](https://img.shields.io/crates/v/elf_loader.svg)](https://crates.io/crates/elf_loader)
[![](https://img.shields.io/crates/d/elf_loader.svg)](https://crates.io/crates/elf_loader)
[![license](https://img.shields.io/crates/l/elf_loader.svg)](https://crates.io/crates/elf_loader)
# elf_loader
一个提供异步加载接口，能够从内存、文件加载elf动态库的Rust库。  

[文档](https://docs.rs/elf_loader/)

# 特性
### ✨ 可以在 `no_std` 环境中工作 ✨
此包提供了一个不使用任何 std 特性的 elf 加载接口，因此可以在内核和嵌入式设备等`no_std`环境中使用。

### ✨ 速度快 ✨
该crate充分利用了rust的一些特性，可以生成性能优异的代码。

### ✨ 非常容易移植，具有良好的可扩展性 ✨
如果您想要移植此 crate，则只需为您的平台实现 `Mmap` 特征即可，并且您可以使用hook函数基于此 crate 实现其他功能。

### ✨ 轻量化 ✨
在使用最少feature的情况下，本库只依赖 `elf`, `cfg-if`, 和 `bitflags` 这额外的三个库。

### ✨ 编译期检查 ✨
利用Rust的生命周期机制，在编译期检查动态库的依赖库是否被提前销毁，以及符号所属的动态库是否已经被销毁。  
比如说有三个被`elf_loader`加载的动态库`a`,`b`,`c`，其中`c`依赖`b`，`b`依赖`a`，如果`a`，`b`中的任意一个在`c` drop之前被drop了，那么将不会程序通过编译。（你可以在[examples/relocate](https://github.com/weizhiao/elf_loader/blob/main/examples/relocate.rs)中验证这一点）

# 用途
它实现了加载elf文件的通用步骤，并留下了扩展接口，用户可以使用它实现自己的定制化loader。

# 特性

| 特性      |  描述  |
| --------- | ----------------- |
| fs        |  启用对文件系统的支持        						|
| use-libc  |  使用libc作为后端，否则直接使用linux syscalls		|
| mmap      |  在加载elf文件时，使用有mmap的平台上的默认实现  	| 
| version   |  在解析符号时使用符号的版本信息     |

# 示例
## 加载一个简单的动态库

```rust
use elf_loader::{Loader, mmap::MmapImpl, object::ElfFile};
use elf_loader::{Loader, mmap::MmapImpl, object::ElfFile};
use std::{collections::HashMap, ptr::null};

fn main() {
    fn print(s: &str) {
        println!("{}", s);
    }

	// liba.so依赖的符号
    let mut map = HashMap::new();
    map.insert("__gmon_start__", null());
    map.insert("__cxa_finalize", null());
    map.insert("_ITM_registerTMCloneTable", null());
    map.insert("_ITM_deregisterTMCloneTable", null());
    map.insert("print", print as _);
    let pre_find = |name: &str| -> Option<*const ()> { map.get(name).copied() };
	// 加载动态库liba.so
	let loader = Loader::<MmapImpl>::new();
    let liba = loader
        .easy_load_dylib(ElfFile::from_path("target/liba.so").unwrap())
        .unwrap();
	// 重定位liba.so中的符号
    let a = liba.easy_relocate([].iter(), &pre_find).unwrap();
	// 调用liba.so中的函数a
    let f = unsafe { a.get::<fn() -> i32>("a").unwrap() };
    f();
}
```

## mini-loader
本仓库提供了一个使用`elf_loader`实现[mini-loader](https://github.com/weizhiao/elf_loader/tree/main/mini-loader)的例子。miniloader可以加载pie文件，目前只支持`x86_64`。  

加载ls:
```shell 
$ cargo build --release -p mini-loader --target=x86_64-unknown-none
$ ./mini-loader /bin/ls
```
需要注意的是必须使用release参数编译mini-loader。

## dlopen-rs
[dlopen-rs](https://crates.io/crates/dlopen-rs)也是基于`elf_loader`库实现的。它实现了dlopen的功能，可以在运行时打开动态库。

# 未完成
* 支持更多的CPU指令集（目前只支持AArch64，Riscv64，X86-64）。
* 完善对DT_FLAGS标志位的支持。
* 完善注释和文档。  
* 增加示例（比如使用异步接口加载动态库的示例）。
* 为示例mini-loader支持更多的指令集。
* 增加性能测试
.....

# 补充
如果在使用过程中遇到问题可以在 GitHub 上提出问题，十分欢迎大家为本库提交代码一起完善`elf_loader`的功能。😊