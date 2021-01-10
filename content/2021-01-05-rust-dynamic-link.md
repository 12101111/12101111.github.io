+++
title = "Rust动态链接"
date = 2021-01-05
[taxonomies]
categories = ["Rust"]
tags = ["Rust","Linux"]
+++

本文探讨一下Rust的crate类型的概念, 以及如何在Rust中使用动态链接编译动态库. 同时对比rust编译出的动态库和使用C语言接口动态库, 看看rust动态库是否能实现C语言接口动态库的功能.

<!-- more -->

# crate类型与链接

crate type是一个rustc的概念,而不是rust语言或者是cargo的概念. 由于大多数rust项目使用[标准的文件目录结构](https://doc.rust-lang.org/cargo/guide/project-layout.html),而cargo会[自动推测目标的类型](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#target-auto-discovery), 因此并不需要手动设置crate类型. 如果需要的话(例如编写proc-macro), 则需要在`Cargo.toml`中[设置](https://doc.rust-lang.org/cargo/reference/cargo-targets.html).

此外,不使用cargo时, 通过rustc也可以指定crate类型, 一种方式是命令行, 例如`rustc --crate-type=bin`, 另一种是`crate_type`属性(built-in attribute), 需要指定在文档的开头, 例如`!#[crate_type = "bin"]`.如果二者同时使用, 命令行指定的值优先.实际上rustc正是使用命令行参数告诉rustc crate的类型.

rustc支持7种crate类型, 在源码中, crate类型是定义在`enum CrateType`中的([cargo的类型定义](https://github.com/rust-lang/cargo/blob/master/src/cargo/core/compiler/crate_type.rs), [rustc的类型定义](https://github.com/rust-lang/rust/blob/master/compiler/rustc_session/src/config.rs#L707))

[rust reference](https://doc.rust-lang.org/reference/linkage.html),给出了这7种类型的区别.

## `bin`

`bin` crate 会编译为一个可运行的二进制文件.`bin`类型不能作为其他crate的依赖, 并且通常会编译为一个可以运行的二进制程序, 通常包含一个`main`函数作为开始运行的入口点. 在上一篇文章我介绍了一些奇怪没有`main`函数的`bin`类型的crate.编译脚本([Build Scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html), 通常为build.rs)也是一个单独的`bin` crate, 并在编译主crate前编译并运行.

## `lib`

`lib` crate泛指不是`bin`类型的crate, 其不会编译为二进制程序, 且不必包含`main`函数(你要愿意的话, 你可以写一个main函数, 仅仅是一个叫`mian`的函数而已), 且可以作为其他crate的依赖.

至于最终`lib` crate会编译出什么, 则会根据依赖它的crate决定.

### `rlib`

`rlib`即rust静态库, 这是一个rustc和cargo使用的中间格式, 虽然和Unix archive(ar文件)格式相同, 但其中的存储的目标文件(`.o`)包含了rustc生成的元数据(rmeta), 根据这些元数据, rlib可以链接到其他rust库, 并产生其他格式的输出.`rlib`并不包括其依赖的代码, rustc会通过rlib中的元数据查找依赖.

### `dylib`

dylib用于将该依赖编译为动态库, 这样依赖其的`bin`类型crate会在运行前由操作系统的动态库加载器加载该依赖.其产生的文件扩展名在Linux上为`so`, Windows上为`dll`, macOS/iOS上为`dylib`, 且Linux的文件名具有`lib`的前缀.

我们随后会发现rust生成的dylib和C语言库略有不同, 这导致其用法也发生了很大的变化.在目前, `dylib`几乎只能作为cargo编译的中间产物使用, 就像rlib一样.

## `cdylib`

cdylib是一种动态库, 与`dylib`不同的是, cdylib是为其他语言提供的动态库, 因此cdylib会静态链接rust的标准库, 以及递归的静态链接该crate的所有依赖, 这样输出的动态库只会暴露库所声明的符号, 不会依赖标准库的符号.`cdylib`和`bin`类型一样, 是最终产物, 因此不能作为rust的依赖出现.`cdylib`会最终作为其他语言(如C/C++)编写的可执行程序的动态库依赖, 或者被其他语言(如Python)以`dlopen`等动态加载的方式加载.cdylib输出的扩展名与dylib相同.

## `staticlib`

`staticlib`是一种静态库, 与`rlib`不同的是, `staticlib`是为其他语言提供的静态库, 因此和`cdylib`一样, `staticlib`会会静态链接rust的标准库, 以及递归的静态链接该crate的所有依赖.其产生的文件扩展名在Linux和macOS上为`.a`, 在Windows上为`.lib`, 且Linux的文件名具有`lib`的前缀.

### proc-macro

`proc-macro`是[过程宏](https://doc.rust-lang.org/reference/procedural-macros.html)编译的产物, 其总是会静态链接到`libstd`, `proc_macro`, 且只能以和`rustc`相同的架构编译.除去链接的特殊性, `proc-macro`和cdylib非常类似.

在`proc_macro`crate中有一套基于C ABI的服务端(rustc前端)和客户端(即proc-macro动态库)相互通信的设施, 这样即使二者使用不同的rust ABI, 只要来自于同一份代码(即源码级兼容), 就可以相互调用.

## rlib or dylib

可以看到, 实际上只有rlib和dylib两种库可以作为其他rust crate的依赖形式.与c语言风格的静态库(staticlib)和动态库(cdylib)不同的是, rlib和dylib虽然仍是Unix archive和so文件, 但是嵌入了rustc的meta data(rmeta文件)

rustc采用一种复杂的手段确定链接一个rust依赖的哪一种形式的库.(我们这里只讨论rust的库和依赖, C依赖只能通过build.rs修改LDFLAGS链接, 因此不做讨论)

1. 如果要产生一个staticlib, 那么其依赖只能使用`rlib`库, 因此已经编译成为`dylib`后就不能再转换为`rlib`了. staticlib是一个最终输出的目标, 因此该crate的所有递归依赖都需要是rlib, 如果不满足该条件, 则编译失败.
2. 如果要产生一个rlib, 那么其依赖可以是rlib或dylib. 这时候这个crate实际上是作为其他crate的依赖(rlib本身并没有什么用处)
3. 如果要产生一个bin, 且没有设置`-C prefer-dynamic`选项, 则首先使用rlib编译其依赖(即默认为静态链接), 如果其依赖不能作为rlib编译(例如某个依赖设置`crate_type=dylib), 则需要同时混合rlib和dylib.
4. 如果要生成dylib, cdylib, 非静态链接的bin, 那么就需要混合链接dylib和rlib, 此时会遇到一个问题: 如果依赖图中2个dylib动态链接1个dylib, 这时合法的, 但是如果2个dylib静态链接同1个rlib, 随后再动态链接这两个dylib时就会出错, 因为这两个dylib提供了重复的符号.这时rustc会使用贪心算法判断将一个依赖编译为rlib还是dylib, 其策略为: 被两个dylib依赖的也是dylib, 其他的是rlib.

https://doc.rust-lang.org/nightly/nightly-rustc/rustc_metadata/dependency_format/index.html

## 链接是一件复杂的事情

虽然Cargo给用户提供了一个友好的界面来管理依赖, 但背后的rustc却隐藏着复杂的细节. 这使得rustc相较于C编译器更加复杂.

在C/C++编译器中:

- 编译一个动态库: `gcc -shared -o libabc.so abc.c`
- 编译一个静态库: `ar rcs libabc.a a.o b.o c.o`
- 依赖一个动态库: `gcc -o xyz -L/usr/local/lib -labc xyz.c`
- 依赖一个静态库: `gcc -o xyz libabc.a xyz.c`

C语言使用动态库的主要动机是

1. 多个进程依赖同一个动态库时, 内存中只需要有一个动态库即可
2. 动态库更新时不需要重新编译依赖它的程序

我们随后会看到, rust至多只能实现1, 而2则是不能实现的.

## cargo和rustc的输出文件

要想研究如何动态链接rust库, 我们先看看正常情况下

我们新建一个bin crate: `cargo new hello`和一个lib crate: `cargo new --lib mylib`, 将`mylib`添加到`hello`的依赖中: `mylib = { path="../mylib" }`

编译`hello`, 看看cargo为我们生成了什么

```output
> cd hello
> cargo build -v
   Compiling mylib v0.1.0 (/tmp/rust/mylib)
     Running `rustc --crate-name mylib --edition=2018 /tmp/rust/mylib/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 -C metadata=ff19b248b0012d99 -C extra-filename=-ff19b248b0012d99 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps -C target-feature=-crt-static`
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hello --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=0cf1e5265f244c28 -C extra-filename=-0cf1e5265f244c28 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib-ff19b248b0012d99.rlib -C target-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.39s
> exa -T target --ignore-glob incremental
target
├── CACHEDIR.TAG
└── debug
   ├── build
   ├── deps
   │  ├── hello-0cf1e5265f244c28
   │  ├── hello-0cf1e5265f244c28.d
   │  ├── libmylib-ff19b248b0012d99.rlib
   │  ├── libmylib-ff19b248b0012d99.rmeta
   │  └── mylib-ff19b248b0012d99.d
   ├── examples
   ├── hello
   └── hello.d
```

`incremental`目录存放了增量式编译所存放的依赖图, 查询缓存, 编译出来的目标文件等文件, 通过rustc的`-C incremental=<PATH>`选项生成

d文件是纯文本文件, 使用类似makefile的格式记录了源码和中间文件的依赖关系, 通过rustc的`--emit=dep-info`选项生成.

例如 `target/debug/deps/hello-0cf1e5265f244c28.d`的内容:

```makefile
/tmp/rust/hello/target/debug/deps/hello-0cf1e5265f244c28: src/main.rs
  
/tmp/rust/hello/target/debug/deps/hello-0cf1e5265f244c28.d: src/main.rs

src/main.rs:
```

`target/debug/deps/mylib-ff19b248b0012d99.d`:

```makefile
/tmp/rust/hello/target/debug/deps/mylib-ff19b248b0012d99.rmeta: /tmp/rust/mylib/src/lib.rs
  
/tmp/rust/hello/target/debug/deps/libmylib-ff19b248b0012d99.rlib: /tmp/rust/mylib/src/lib.rs

/tmp/rust/hello/target/debug/deps/mylib-ff19b248b0012d99.d: /tmp/rust/mylib/src/lib.rs

/tmp/rust/mylib/src/lib.rs:
```

`target/debug/hello.d`:

```makefile
/tmp/rust/hello/target/debug/hello: /tmp/rust/hello/src/main.rs /tmp/rust/mylib/src/lib.rs
```

rmeta文件是二进制的crate元数据文件, 通过rustc的`--emit=metadata`选项生成.

libmylib-ff19b248b0012d99.rlib是mylib这个库的最终编译出来的二进制文件, 由于cargo默认生成的lib crate只有一个单元测试, 因此该文件并不包括任何代码.前文提到, rlib实际上是Unix archive,因此可以使用readelf查看, 对于该rlib文件, 其中包括一个目标文件: `mylib-ff19b248b0012d99.1nemtrzp8kgtsf6b.rcgu.o`

```output
> readelf -a target/debug/deps/libmylib-ff19b248b0012d99.rlib

File: target/debug/deps/libmylib-ff19b248b0012d99.rlib(mylib-ff19b248b0012d99.1nemtrzp8kgtsf6b.rcgu.o)
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          304 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         7
  Section header string table index: 1
There are 7 section headers, starting at offset 0x130:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 0000b0 00007e 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 000000 00  AX  0   0  4
  [ 3] .debug_gdb_scripts PROGBITS       0000000000000000 000040 000022 01 AMS  0   0  1
  [ 4] .debug_aranges    PROGBITS        0000000000000000 000062 000000 00      0   0  1
  [ 5] .note.GNU-stack   PROGBITS        0000000000000000 000062 000000 00      0   0  1
  [ 6] .symtab           SYMTAB          0000000000000000 000068 000048 18      1   2  8
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

Elf file type is REL (Relocatable file)
Entry point 0x0
There are 0 program headers, starting at offset 0

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align

 Section to Segment mapping:
  Segment Sections...
   None   .strtab .text .debug_gdb_scripts .debug_aranges .note.GNU-stack .symtab

There are no relocations in this file.

Symbol table '.symtab' contains 3 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS 1nemtrzp8kgtsf6b
     2: 0000000000000000    34 OBJECT  WEAK   DEFAULT     3 __rustc_debug_gdb_scripts_section__
There are no section groups in this file.
```

hello-0cf1e5265f244c28就是最终的二进制可执行程序, 观察第二个rustc的命令行, 可以发现rust的crate依赖是通过rustc的`--extern <crate name>=<path to crate rlib>`传递的.

deps目录中的16位ASCII后缀就是crate的metadata, 由cargo通过rustc的`-C metadata=<HASH> -C extra-filename=-<HASH>`选项传递.该hash包括了package id(crate 名称, 版本号, url), crate启用的features, 依赖的metadata, 编译时的profile和mode(如unwind设置, 优化级别, debug info 等), target名称和类型(本机编译还是交叉编译), rustc版本号.更多信息参考[计算metadata hash的cargo源码](https://github.com/rust-lang/cargo/blob/2f218d0d1b3724d8ce9b3868f6b18d4b2ce5c9a5/src/cargo/core/compiler/context/compilation_files.rs#L489)

我们注意到, 变更rustflags不会影响metadata的hash结果, 在不更改package id的情况下修改源代码也不会影响hash.而一旦一个crate的hash发生了变化, 所有依赖这个crate的hash也会变化.需要注意的是cargo并非通过metadata判断是否重编译crate, 而是通过[fingerprint](https://github.com/rust-lang/cargo/blob/master/src/cargo/core/compiler/fingerprint.rs)决定是否重编译.

有关更多cargo是如何调用rustc的信息, 可以查看[cargo的源代码](https://github.com/rust-lang/cargo/blob/2f218d0d1b3724d8ce9b3868f6b18d4b2ce5c9a5/src/cargo/core/compiler/mod.rs#L771)

更改两个crate的代码, 使得`hello`用到`mylib`的代码.

`mylib/src/lib.rs`:

```rust
pub fn num() -> i32 {
    1
}
```

`hello/src/main.rs`:

```rust
use mylib::num;
fn main() {
    println!("Hello, {} world!", num());
}
```

再次编译并运行:

```output
> cargo build -v
   Compiling mylib v0.1.0 (/tmp/rust/mylib)
     Running `rustc --crate-name mylib --edition=2018 /tmp/rust/mylib/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 -C metadata=ff19b248b0012d99 -C extra-filename=-ff19b248b0012d99 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps -C target-feature=-crt-static`
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hello --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=0cf1e5265f244c28 -C extra-filename=-0cf1e5265f244c28 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib-ff19b248b0012d99.rlib -C target-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.30s
> ./target/debug/hello
Hello, 1 world!
```

可见更改代码的确不会变动metadata的hash值.

## 动态链接

```
> RUSTFLAGS="-C prefer-dynamic -C target-feature=-crt-static" cargo build -v
   Compiling mylib v0.1.0 (/tmp/rust/mylib)
     Running `rustc --crate-name mylib --edition=2018 /tmp/rust/mylib/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type lib --emit=dep-info,metadata,link -C embed-bitcode=no -C debuginfo=2 -C metadata=ff19b248b0012d99 -C extra-filename=-ff19b248b0012d99 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps -C prefer-dynamic -C target-feature=-crt-static`
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hello --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=0cf1e5265f244c28 -C extra-filename=-0cf1e5265f244c28 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib-ff19b248b0012d99.rlib -C prefer-dynamic -C target-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
> exa -T target --ignore-glob incremental
target
├── CACHEDIR.TAG
└── debug
   ├── build
   ├── deps
   │  ├── hello-0cf1e5265f244c28
   │  ├── hello-0cf1e5265f244c28.d
   │  ├── libmylib-ff19b248b0012d99.rlib
   │  ├── libmylib-ff19b248b0012d99.rmeta
   │  └── mylib-ff19b248b0012d99.d
   ├── examples
   ├── hello
   └── hello.d
> ./target/debug/hello
Error loading shared library libstd-aef788a827ed39d9.so: No such file or directory (needed by ./target/debug/hello)
Error relocating ./target/debug/hello: _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE: symbol not found
Error relocating ./target/debug/hello: rust_eh_personality: symbol not found
Error relocating ./target/debug/hello: _ZN3std2io5stdio6_print17h1ded803b1aed6cccE: symbol not found
Error relocating ./target/debug/hello: _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E: symbol not found
> readelf -d target/debug/hello | grep NEEDED
  0x0000000000000001 (NEEDED)       Shared library: [libstd-aef788a827ed39d9.so]
  0x0000000000000001 (NEEDED)       Shared library: [libgcc_s.so.1]
  0x0000000000000001 (NEEDED)       Shared library: [libc.so]
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib ./target/debug/hello
Hello, 1 world!
```

这时生成的二进制程序依赖dylib版的libstd.根据前文提到的rustc决定依赖格式(也就是当`crate-type=lib`时`-C emit=link`时生成dylib还是rlib)的逻辑, 由于libstd是预编译且同时提供了rlib和dylib两种格式, 因为prefer-dynamic开关的存在, 选择dylib版的libstd, 而`mylib`没有编译(fingerprint发生变化,原来的编译文件无效), 既可以使用rlib也可以使用dylib, 且不依赖dylib, 因此自动选择了rlib, 因此 `mylib` 并没有编译为dylib.

查看std的`Cargo.toml`可以得知, 在`mylib`的`Cargo.toml`中加入

```toml
[lib]
crate-type = ["dylib", "rlib"]
```

可以使得该crate可以编译为dylib或rlib, 且在prefer-dynamic存在时主动选择dylib

重编译`hello`并运行

```output
> RUSTFLAGS="-C prefer-dynamic -C target-feature=-crt-static" cargo build -v
   Compiling mylib v0.1.0 (/tmp/rust/mylib)
     Running `rustc --crate-name mylib --edition=2018 /tmp/rust/mylib/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type dylib --crate-type rlib --emit=dep-info,link -C prefer-dynamic -C embed-bitcode=no -C debuginfo=2 -C metadata=b6125a00ae42b63c --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps -C prefer-dynamic -C target-feature=-crt-static`
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hello --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=29d11507098fa904 -C extra-filename=-29d11507098fa904 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.so --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.rlib -C prefer-dynamic -C target-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
> exa -T target --ignore-glob incremental
target
├── CACHEDIR.TAG
└── debug
   ├── build
   ├── deps
   │  ├── hello-29d11507098fa904
   │  ├── hello-29d11507098fa904.d
   │  ├── libmylib.rlib
   │  ├── libmylib.so
   │  └── mylib.d
   ├── examples
   ├── hello
   ├── hello.d
   └── libmylib.so
> ./target/debug/hello
Error loading shared library libmylib.so: No such file or directory (needed by ./target/debug/hello)
Error loading shared library libstd-aef788a827ed39d9.so: No such file or directory (needed by ./target/debug/hello)
Error relocating ./target/debug/hello: rust_eh_personality: symbol not found
Error relocating ./target/debug/hello: _ZN3std2io5stdio6_print17h1ded803b1aed6cccE: symbol not found
Error relocating ./target/debug/hello: _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E: symbol not found
Error relocating ./target/debug/hello: _ZN5mylib3num17hefef8e0aa2fb2876E: symbol not found
Error relocating ./target/debug/hello: _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE: symbol not found
> readelf -d target/debug/hello | grep NEEDED
  0x0000000000000001 (NEEDED)       Shared library: [libmylib.so]
  0x0000000000000001 (NEEDED)       Shared library: [libstd-aef788a827ed39d9.so]
  0x0000000000000001 (NEEDED)       Shared library: [libgcc_s.so.1]
  0x0000000000000001 (NEEDED)       Shared library: [libc.so]
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug ./target/debug/hello
Hello, 1 world!
```

(为什么在这里不使用cargo run? 因为使用cargo run可以直接运行, 不会报错找不到共享库文件, cargo设置了LD_LIBRARY_PATH, 但是并没有在输出中体现出来)

值得注意的是dylib的编译结果的文件名并不包括metadata hash, 但libstd因为并不是标准的cargo包因此较为特殊.

## 动态链接的rust程序能否共享依赖

```
> mkdir src/bin
> cp src/main.rs src/bin/hi.rs
```

把`hi.rs` 稍作修改: `println!("Hello, {} world!", num() * 2);`

```shell
> cargo clean
> RUSTFLAGS="-C prefer-dynamic -C target-feature=-crt-static" cargo build -v --bin hello
   Compiling mylib v0.1.0 (/tmp/rust/mylib)
     Running `rustc --crate-name mylib --edition=2018 /tmp/rust/mylib/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type dylib --crate-type rlib --emit=dep-info,link -C prefer-dynamic -C embed-bitcode=no -C debuginfo=2 -C metadata=b6125a00ae42b63c --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps -C prefer-dynamic -C target-feature=-crt-static`
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hello --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=29d11507098fa904 -C extra-filename=-29d11507098fa904 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.so --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.rlib -C prefer-dynamic -C target-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
> RUSTFLAGS="-C prefer-dynamic -C target-feature=-crt-static" cargo build -v --bin hi
       Fresh mylib v0.1.0 (/tmp/rust/mylib)
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hi --edition=2018 src/bin/hi.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=24d23b4e455b3276 -C extra-filename=-24d23b4e455b3276 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.so --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.rlib -C prefer-dynamic -C target-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.16s
> exa -T target --ignore-glob incremental
target
├── CACHEDIR.TAG
└── debug
   ├── build
   ├── deps
   │  ├── hello-29d11507098fa904
   │  ├── hello-29d11507098fa904.d
   │  ├── hi-24d23b4e455b3276
   │  ├── hi-24d23b4e455b3276.d
   │  ├── libmylib.rlib
   │  ├── libmylib.so
   │  └── mylib.d
   ├── examples
   ├── hello
   ├── hello.d
   ├── hi
   ├── hi.d
   └── libmylib.so
> export LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug
> ./target/debug/hello
Hello, 1 world!
> ./target/debug/hi
Hello, 2 world!
```

注意到编译`hi`时, 输出的`Fresh mylib v0.1.0 (/tmp/rust/mylib)`表明cargo并没有重编译`mylib`.
最终这两个bin都可以正常运行, 我们看看它们是不是真的依赖libmylib.so

```output
> readelf -d target/debug/hello | grep NEEDED
  0x0000000000000001 (NEEDED)       Shared library: [libmylib.so]
  0x0000000000000001 (NEEDED)       Shared library: [libstd-aef788a827ed39d9.so]
  0x0000000000000001 (NEEDED)       Shared library: [libgcc_s.so.1]
  0x0000000000000001 (NEEDED)       Shared library: [libc.so]
> readelf -d target/debug/hi | grep NEEDED
  0x0000000000000001 (NEEDED)       Shared library: [libmylib.so]
  0x0000000000000001 (NEEDED)       Shared library: [libstd-aef788a827ed39d9.so]
  0x0000000000000001 (NEEDED)       Shared library: [libgcc_s.so.1]
  0x0000000000000001 (NEEDED)       Shared library: [libc.so]
```

至少elf头中记录了hello和hi都需要`libmylib.so`和`libstd-aef788a827ed39d9.so`, 我们看看符号是否能对的上.使用nm工具查看3个elf的符号:

```output
> nm target/debug/libmylib.so
0000000000002640 d _DYNAMIC
00000000000015e0 T _ZN5mylib3num17hefef8e0aa2fb2876E
00000000000004e0 r __EH_FRAME_LIST_END__
                 w __cxa_finalize
                 w __deregister_frame_info
00000000000037f8 d __dso_handle
                 w __register_frame_info
0000000000000498 r __rustc_debug_gdb_scripts_section__
00000000000015eb t _fini
00000000000015e8 t _init
0000000000000000 N rust_metadata_mylib_ee167d4ab2087d981bbef06aea3f858f
> nm target/debug/hello
00000000000032e8 d DW.ref.rust_eh_personality
00000000000008ac r GCC_except_table0
00000000000008a0 r GCC_except_table2
0000000000002060 d _DYNAMIC
                 U _Unwind_Resume
0000000000001e80 t _ZN3std10sys_common9backtrace28__rust_begin_short_backtrace17h1532b767e1a19115E
                 U _ZN3std2io5stdio6_print17h1ded803b1aed6cccE
0000000000001ec0 t _ZN3std2rt10lang_start17h393e70d55468c5deE
0000000000001f20 t _ZN3std2rt10lang_start28_$u7b$$u7b$closure$u7d$$u7d$17h7c2fd3be77659b30E
                 U _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE
0000000000001f90 t _ZN3std3sys4unix7process14process_common8ExitCode6as_i3217h7a7beeee9da95e03E
0000000000001d00 t _ZN4core3fmt10ArgumentV13new17h58403316e9e3aed8E
                 U _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E
0000000000001d60 t _ZN4core3fmt9Arguments6new_v117h641b32bcb1583909E
0000000000001c80 t _ZN4core3ops8function6FnOnce40call_once$u7b$$u7b$vtable.shim$u7d$$u7d$17hebed9ec320856400E
0000000000001ca0 t _ZN4core3ops8function6FnOnce9call_once17hd6f7e85eff99597aE
0000000000001cb0 t _ZN4core3ops8function6FnOnce9call_once17he86a1358c035a351E
0000000000001cf0 t _ZN4core3ptr13drop_in_place17hdc22de41e16c2f90E
0000000000001db0 t _ZN4core4hint9black_box17h2501d3e9821f440dE
0000000000001f50 t _ZN54_$LT$$LP$$RP$$u20$as$u20$std..process..Termination$GT$6report17h8dc214f9ad10d68aE
0000000000001dc0 t _ZN5hello4main17h080ec7908287a1a0E
                 U _ZN5mylib3num17hefef8e0aa2fb2876E
0000000000001f70 t _ZN68_$LT$std..process..ExitCode$u20$as$u20$std..process..Termination$GT$6report17hc95702f5648ae8a0E
0000000000000988 r __EH_FRAME_LIST_END__
                 w __cxa_finalize
                 w __deregister_frame_info
00000000000032e0 d __dso_handle
                 U __libc_start_main
                 w __register_frame_info
00000000000008bc V __rustc_debug_gdb_scripts_section__
0000000000001f9d T _fini
0000000000001f9a T _init
0000000000001ba0 T _start
0000000000001bc0 T _start_c
0000000000001e50 T main
                 U rust_eh_personality
> nm target/debug/hi
00000000000043e8 d DW.ref.rust_eh_personality
0000000000000950 r GCC_except_table0
0000000000000944 r GCC_except_table1
0000000000003158 d _DYNAMIC
                 U _Unwind_Resume
0000000000001e90 t _ZN2hi4main17h9e2072fe35c4b5b4E
0000000000001fe0 t _ZN3std10sys_common9backtrace28__rust_begin_short_backtrace17he1d71d9db80c0530E
                 U _ZN3std2io5stdio6_print17h1ded803b1aed6cccE
0000000000001d80 t _ZN3std2rt10lang_start17hc99a7c63ea252eaeE
0000000000001de0 t _ZN3std2rt10lang_start28_$u7b$$u7b$closure$u7d$$u7d$17h031e9375f8893238E
                 U _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE
0000000000002020 t _ZN3std3sys4unix7process14process_common8ExitCode6as_i3217h7189431738fbb130E
0000000000001d20 t _ZN4core3fmt10ArgumentV13new17h86fc128822eafca9E
                 U _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E
0000000000002030 t _ZN4core3fmt9Arguments6new_v117h71ce12250f51dee7E
0000000000001e10 t _ZN4core3ops8function6FnOnce40call_once$u7b$$u7b$vtable.shim$u7d$$u7d$17h98204e361d18157eE
0000000000001e30 t _ZN4core3ops8function6FnOnce9call_once17h69c2245f11c57f2cE
0000000000001e70 t _ZN4core3ops8function6FnOnce9call_once17h708925e4b9276646E
0000000000001e80 t _ZN4core3ptr13drop_in_place17h0ea2467bdcaed3a3E
0000000000001f90 t _ZN4core4hint9black_box17h9c97f0f64843d9bbE
                 U _ZN4core9panicking5panic17h1c3c966b042834b6E
0000000000001fa0 t _ZN54_$LT$$LP$$RP$$u20$as$u20$std..process..Termination$GT$6report17h5775a3587e9f1ccaE
                 U _ZN5mylib3num17hefef8e0aa2fb2876E
0000000000001fc0 t _ZN68_$LT$std..process..ExitCode$u20$as$u20$std..process..Termination$GT$6report17h37da287fc237aeb0E
0000000000000a38 r __EH_FRAME_LIST_END__
                 w __cxa_finalize
                 w __deregister_frame_info
00000000000043e0 d __dso_handle
                 U __libc_start_main
                 w __register_frame_info
0000000000000920 V __rustc_debug_gdb_scripts_section__
000000000000207b T _fini
0000000000002078 T _init
0000000000001c40 T _start
0000000000001c60 T _start_c
0000000000001f60 T main
                 U rust_eh_personality
0000000000000980 r str.0
```

这样看起来太乱了, 我们只关系so动态库定义的全局text段符号(T),和可执行程序未定义的符号(U)

```
> nm target/debug/hi | grep U
                 U _Unwind_Resume
                 U _ZN3std2io5stdio6_print17h1ded803b1aed6cccE
                 U _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE
                 U _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E
                 U _ZN4core9panicking5panic17h1c3c966b042834b6E
                 U _ZN5mylib3num17hefef8e0aa2fb2876E
                 U __libc_start_main
                 U rust_eh_personality
> nm target/debug/hello| grep U
                 U _Unwind_Resume
                 U _ZN3std2io5stdio6_print17h1ded803b1aed6cccE
                 U _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE
                 U _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E
                 U _ZN5mylib3num17hefef8e0aa2fb2876E
                 U __libc_start_main
                 U rust_eh_personality
> nm target/debug/libmylib.so| grep T
00000000000015e0 T _ZN5mylib3num17hefef8e0aa2fb2876E
00000000000004e0 r __EH_FRAME_LIST_END__
```

`libmylib.so`提供的`_ZN5mylib3num17hefef8e0aa2fb2876E`符号的却是hello和hi未定义的符号, 非常正确.

用objdump查看该符号的反汇编

```s
00000000000015e0 <_ZN5mylib3num17hefef8e0aa2fb2876E>:
    15e0: b8 01 00 00 00               	movl	$1, %eax
    15e5: c3                           	retq
    15e6: cc                           	int3
    15e7: cc                           	int3
```

的确是我们要的那个函数.

到此我们可以做出结论, 使用dylib的确可以使其在多个rust bin之间共享, 只需要设置`crate-type = ["dylib", ...]`即可.

实际上, rustc的主要部分位于`librustc_driver-<hash>.so`中, 其大小为数十到数百MB, 而`rustc`可执行程序仅为数MB, 只是一个命令行前端而已.而`librustc_driver`则被rustc, clippy-driver, rls, rustdoc等程序共享, 减小了整个编译器工具链的大小.

但是, 非常明显的问题就是, 我们不能让所有的crate动态链接, 只有那些在Cargo.toml特别设置的crate才能编译为dylib.这是一个小小的局限性.

## 动态升级

另一个动态库的特性是动态库升级后, 如果没有发生break change导致ABI变化, 依赖其的程序/其他动态库不需要重编译.在Unix系统上, 这一机制通常使用soname实现. 例如webkit-gtk这个c/c++编写的浏览器引擎, 其文件为`/usr/lib/libwebkit2gtk-4.0.so.37.49.8`, 并且有符号链接`/usr/lib/libwebkit2gtk-4.0.so`和`/usr/lib/libwebkit2gtk-4.0.so.37`, 这里soname就是`.so`后面的`37.49.8`.webkit-gtk发生一个A BI稳定的升级后, 其soname可能增加, 例如变为`37.49.9`或`37.50.0`, 但依赖webkit-gtk的程序会使用`libwebkit2gtk-4.0.so.37`这个符号链接, 这样不需要修改程序, 就会自动调用新的webkit-gtk.

ABI是应用程序二进制接口(application binary interface)的缩写, 和API(application programming interface)类似又不同.API 是一个程序或库对开发者提供的一组接口, 包括了函数, 常量, 全局变量, 结构体, 枚举的定义, 这样开发者可以在不了解函数实现的前提下使用该程序和库.通常API稳定是指现有的函数名和签名不发生变化, 常量和全局变量名称不变, 结构体现有各个成员的名称和类型不变, 枚举现有各个成员的名称和类型不变, 且前面所述不变的各个项目语义没有发生变化. API稳定是源码级的, 即API稳定的库在升级后, 重新编译依赖其的程序, 就可以继续使用, 无需修改程序.而ABI稳定则更加严格, 常量的类型和值不能发生变化, 函数的符号和调用约定不能发生变化, 全局变量的符号和类型不能发生变化, 包括结构体和枚举在内的各个类型的类型大小、布局、对齐不能发生变化.

大家可能已经知道了, rust并没有一个稳定的ABI. 即使是定义完全相同的struct, 其布局也可能发生变化.

例如一个模拟标准库里的Vec:

```rust
struct MyVec{
  ptr: Unique<T>,
  cap: usize,
  len: usize
}
```

实际上ptr, cap和len的顺序并不是声明的顺序, 且ptr和cap之间可能并不是连续的, 可能存在8 bytes的填充.而这个填充也可能在cap和len之间, 编译器会根据实际各个字段的使用情况对布局进行优化.正是因为这种不确定性, 一方面rustc会生成效率更高的代码,另一方面rust很难在不限制优化水平的情况下制定一个稳定的ABI. 实际上rust所有支持的调用约定中, 只有C调用约定是稳定的, 但这时布局优化就会时效, 很可能产生对内存和缓存不友好的代码.rust nomicon对rust默认的布局做出了解释: [repr(Rust)](https://doc.rust-lang.org/nomicon/repr-rust.html), rust reference则对[类型布局](https://doc.rust-lang.org/reference/type-layout.html)有更详细的解释.

要使用C调用约定, 可以使用`#[repr(C)]`修饰结构体, 使用C语言函数调用约定的函数: `extern "C" fn"`. nomicon也有详细的解释: [ffi](https://doc.rust-lang.org/nomicon/ffi.html)

使用c调用固然可以实现稳定的ABI, 我们研究一下在默认的rust调用约定下会发生什么事情.

我们把上面的hello和hi复制出来, 然后修改`mylib::num`, 使其返回2, 再重新编译, 看看会发生什么

```output
> cp target/debug/hello .
> cp target/debug/hi .
> cp target/debug/libmylib.so .
> ls
Cargo.lock  Cargo.toml  hello  hi  libmylib.so  src  target
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:. ./hello
Hello, 1 world!
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:. ./hi
Hello, 2 world!
> RUSTFLAGS="-C prefer-dynamic -Ctarget-feature=-crt-static" cargo build -v
   Compiling mylib v0.1.0 (/tmp/rust/mylib)
     Running `rustc --crate-name mylib --edition=2018 /tmp/rust/mylib/src/lib.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type dylib --crate-type rlib --emit=dep-info,link -C prefer-dynamic -C embed-bitcode=no -C debuginfo=2 -C metadata=b6125a00ae42b63c --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps -C prefer-dynamic -Ctarget-feature=-crt-static`
   Compiling hello v0.1.0 (/tmp/rust/hello)
     Running `rustc --crate-name hi --edition=2018 src/bin/hi.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=24d23b4e455b3276 -C extra-filename=-24d23b4e455b3276 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.so --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.rlib -C prefer-dynamic -Ctarget-feature=-crt-static`
     Running `rustc --crate-name hello --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi --crate-type bin --emit=dep-info,link -C embed-bitcode=no -C debuginfo=2 -C metadata=29d11507098fa904 -C extra-filename=-29d11507098fa904 --out-dir /tmp/rust/hello/target/debug/deps -C incremental=/tmp/rust/hello/target/debug/incremental -L dependency=/tmp/rust/hello/target/debug/deps --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.so --extern mylib=/tmp/rust/hello/target/debug/deps/libmylib.rlib -C prefer-dynamic -Ctarget-feature=-crt-static`
    Finished dev [unoptimized + debuginfo] target(s) in 0.33s
> exa -T target --ignore-glob incremental
target
├── CACHEDIR.TAG
└── debug
   ├── build
   ├── deps
   │  ├── hello-29d11507098fa904
   │  ├── hello-29d11507098fa904.d
   │  ├── hi-24d23b4e455b3276
   │  ├── hi-24d23b4e455b3276.d
   │  ├── libmylib.rlib
   │  ├── libmylib.so
   │  └── mylib.d
   ├── examples
   ├── hello
   ├── hello.d
   ├── hi
   ├── hi.d
   └── libmylib.so
```

注意到这3个crate都重编译了, 我们现在有两份它们的产物, 看看它们的运行结果是什么.

旧libmylib.so+旧程序的结果:

```
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:. ./hello
Hello, 1 world!
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:. ./hi
Hello, 2 world!
```

旧libmylib.so+新程序, 结果和旧libmylib.so+旧程序一样

```
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:. ./target/debug/hello
Hello, 1 world!
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:. ./target/debug/hi
Hello, 2 world!
```

新libmylib.so+新程序的结果:

```
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug ./target/debug/hello
Hello, 2 world!
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug ./target/debug/hi
Hello, 4 world!
```

新libmylib.so+旧程序, 结果和新libmylib.so+新程序一样

```
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug ./hello
Hello, 2 world!
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug ./hi
Hello, 4 world!
```

可见这个修改的确没有改变libmylib.so的ABI.

此时我们将`mylib`的版本号升为0.11, 重新编译, 并运行新libmylib.so+旧程序的组合:

```output
> LD_LIBRARY_PATH=~/.rustup/toolchains/nightly-x86_64-unknown-linux-musl/lib/rustlib/x86_64-unknown-linux-musl/lib:./target/debug ./hi
Error relocating ./hi: _ZN5mylib3num17hefef8e0aa2fb2876E: symbol not found
> nm ./hi | grep U
                 U _Unwind_Resume
                 U _ZN3std2io5stdio6_print17h1ded803b1aed6cccE
                 U _ZN3std2rt19lang_start_internal17h6606fb5e766e2b1cE
                 U _ZN4core3fmt3num3imp52_$LT$impl$u20$core..fmt..Display$u20$for$u20$i32$GT$3fmt17h05aa13cbe62062a9E
                 U _ZN4core9panicking5panic17h1c3c966b042834b6E
                 U _ZN5mylib3num17hefef8e0aa2fb2876E
                 U __libc_start_main
                 U rust_eh_personality
> nm target/debug/libmylib.so
0000000000002640 d _DYNAMIC
00000000000015e0 T _ZN5mylib3num17hcb5835737be2d5afE
00000000000004e0 r __EH_FRAME_LIST_END__
                 w __cxa_finalize
                 w __deregister_frame_info
00000000000037f8 d __dso_handle
                 w __register_frame_info
0000000000000498 r __rustc_debug_gdb_scripts_section__
00000000000015eb t _fini
00000000000015e8 t _init
0000000000000000 N rust_metadata_mylib_1bbf50884b077dacedc7265861b262ff
```

注意到libmylib.so符号名后面的hash发生了变化, 这导致elf无法找到原有的符号.实际上, 这个hash是rust实现单crate多版本共存的重要机制, 被称为SVH (strict version hash), 是整个crate计算得出的hash结果. SVH的计算过程是高度不稳定的rustc内部细节, 我们只能假设当任何crate的依赖, 元数据(例如名称和版本号), 源代码(例如文件名和代码的HIR)发生变化时, SVH就会发生变化, 引起重新编译. 同一crate的不同版本计算可以得到不同的SVH,因此rust允许同时使用同一crate的不同版本, 除非该crate包含了`#[no_mangle]`修饰的符号.

但是SVH限制了我们在动态编译时升级版本的能力.由于版本号的变化, SVH会发生变化, 因此我们在升级动态链接的dylib依赖的版本时也必须重编译所有依赖它的程序.虽然rust使用semver标识向前兼容性, 但是rustc显然不信任人类手动标识的版本号, 而是一刀切的认为所有版本号的变化都是breaking change.而且这也包括rustc本身版本号的变化, 在升级rustc后, 即使是隔着一天的nightly, 所有先前编译的二进制文件都会失效.

到目前为止, 唯一可行的方法就是使用cdylib, 将API都导出为C接口, 这样就可以做到不需要重编译的升级库版本了.目前gtk的依赖, 解析并渲染SVG矢量图的库librsvg就是一个rust编写并导出C API的库, 其维护者使用rust的生态重写了librsvg, 并使其C API和ABI没有变化. 但是纯rust生态下, rustc还没有相关的机制或方法实现类似的效果, 我们必须在将动态库和程序一一绑定.
