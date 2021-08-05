+++
title = "使用Rust编写操作系统(一)"
date = 2019-03-22
[taxonomies]
categories = ["OS"]
tags = ["Rust","OS"]
+++

# 前言

Rust语言作为C/C++的安全替代,是21世纪新兴语言中最适合开发操作系统的.

本文介绍有关使用高级语言,Rust,编写操作系统的一些前置知识.

<!-- more -->

使用Rust语言开发的操作系统开源项目有:

[Redox](http://redox-os.org) 接近self-host,具有shell,窗口管理器,文件系统,网络,微内核的复杂项目

[tockos](https://www.tockos.org) 嵌入式OS

[Nebulet](https://github.com/nebulet/nebulet) 一个在ring0运行的WebAssembly虚拟机/系统

[Tifflin](https://github.com/thepowersgang/rust_os)

教育目的的项目:

[清华大学rCore](https://github.com/rcore-os/rCore)

[blog-os](https://os.phil-opp.com/)

[barebones](https://github.com/thepowersgang/rust-barebones-kernel)

什么,你在问cs140e? emmm,那些代码过了不到半年就不能编译了,紧跟nightly才是正道啊.

# 相关知识

## 编程语言

用于操作系统开发的语言最好具有以下功能(否则需要修改编译器进行hack):

- 允许内嵌汇编,或者调用汇编函数.(用于特权指令)
- 允许创建按特定位与字节排序的结构.(例如页表)
- 具有手动管理内存分配的能力.(操作系统需要管理内存分配)
- 具有很小的运行时.(因此不需要过多的汇编或其他语言来引导到该语言)
- 编译到机器码,而不是字节码或解释运行.(同上)
- 编译器需要支持生成操作系统无关的代码,支持交叉编译(否则不能生成内核二进制文件)

现在大多数商业操作系统内核使用C作为内核语言.究其原因,主要是现存商业操作系统内核都诞生于上个世纪,作为早期的高级语言,C语言的移植性极高,复杂度较低,而且用C语言开发的第一个项目就是重写原本是汇编编写的Unix操作系统及程序,使其成为可移植的操作系统,因此习惯上使用C编写操作系统.

另一个可选的语言是C++,目前Google Fuchsia使用C++开发内核,Windows的部分驱动程序也是C++开发的.C++在操作系统开发方面一大问题是其标准委员会不停的往语言中塞入各种功能,其中许多功能是需要操作系统才可以使用的,内核代码不能使用这些功能,比如std命名空间,STL,异常,虚函数,RTTI,dynamic_cast.Fuchsia的内核Zircon的文档[C++ in Zircon](https://fuchsia.googlesource.com/zircon/+/master/docs/cxx.md)介绍了使用C++开发系统内核时的注意事项.

微软曾经试图使用C#开发操作系统,创建了[Singularity](https://en.wikipedia.org/wiki/Singularity_%28operating_system%29)及[Midori](https://en.wikipedia.org/wiki/Midori_(operating_system))实验系统项目,最终项目被砍,一位参加项目的员工在博客中记录了这一系统的细节[(中文翻译)](https://github.com/ZiJing6/blogging-about-midori).虽然证明C#这类托管语言可以作为操作系统的内核,但是这需要强大的开发团队,同时具有编译器和操作系统的开发能力,不是个人开发者能涉及的.

## 编译器

前面提到的各种高级语言,并不能被处理器直接执行.处理器只接受并执行机器码.因此,必须要有合适的编译器将源代码转换成机器码.

高级语言的编译过程一般为:源文件+语言库->编译->汇编代码/IR->汇编器/编译器后端->目标文件->链接->可执行文件.

编译内核时有几处与与普通的程序不同:

### 语言库

大部分库函数不能使用,需要自己实现.常见的有:

    与输入输出有关的:如C的printf,open.内核必须自己实现输入输出的方法.
    与进程/线程有关的.任务管理由内核完成.
    与内存分配有关的.对内存的管理由内核完成.内核在开发/启动初期甚至不能访问堆内存,只能使用栈,或者硬编码的地址.

能够使用的,一般只有基本数据类型相关的库,例如数学计算,字符串处理,memcpy等.

常见的C语言库是[newlib](https://sourceware.org/newlib/libc.html),操作系统开发者只需要移植几个系统调用即可使其他C程序运行.

### 交叉编译

对于GCC编译器,必须重新编译一份交叉编译器,以实现在开发平台上运行编译器,同时生成的目标文件适用于新平台.详见[osdev上的教程](https://wiki.osdev.org/GCC_Cross-Compiler)

### 汇编

仅能被操作系统使用的特权指令一般不能从高级语言翻译过来,必须在高级语言中内嵌或者调用汇编函数以实现对机器的管理功能.

此外,即使是非常简单的C语言,也不能开机就能被CPU直接执行,由于刚开机时内存中的数据为随机的垃圾,因此有必要在调用C语言的第一个函数前使用汇编程序初始化寄存器,初始化bss段,设置栈,随后再跳转到C语言程序的入口.

对于x86＋BIOS这种老的架构,还需要从磁盘加载文件，创建GDT，从实模式切换到保护模式，设置段寄存器等等,这些代码都只能使用汇编编写.

其他语言,如Rust,也需要这种汇编代码.这些代码被称为运行时或者引导程序．

### 链接

对于普通程序,编译的最后一步是链接.链接器会按照程序的格式(Unix常见为ELF文件,macOS使用mach-o文件,Windows使用PE文件)创建程序头,填写各种记录重定向信息的表,然后将目标文件按顺序链接为一个可执行程序,一般还会动态链接系统C语言库.

但是对于操作系统内核而言,其并不是被可执行文件加载器加载,也不支持动态链接,而是被引导程序直接加载到内存中,其格式由硬件固件或引导程序特殊定义.可以使用[ld script](https://sourceware.org/binutils/docs/ld/Scripts.html)将目标文件链接为其特殊格式.

UEFI固件下使用的可引导文件格式为标准的PE文件(即格式与Windows可执行程序相同)，使用GNU工具链的内核常使用ELF格式,并使用引导程序加载.嵌入式设备也使用u-boot作为引导程序,而有些嵌入式设备直接从ROM的某一地址执行.

## Rust

Rust语言提供了对开发操作系统或裸机程序方便的方法.

只需要在lib.rs或者main.rs开头添加`#![no_std]`即可不使用`std`标准库,而是使用平台无关的`core`库.`core`库只包含了基本数据类型及相关的平台无关的函数,可以在任何支持的CPU上运行.

有关不使用`std`的rust程序的示例,可以查看[blog os的第一篇教程](https://os.phil-opp.com/freestanding-rust-binary/).[中文翻译](https://zhuanlan.zhihu.com/p/53064186)

`rustc`具有交叉编译能力,不需要编译`rustc`就可以为其他平台编译代码(但有时你需要合适的链接器).
这种能力来源于`rustc`使用的`llvm`后端.`rustc`默认支持的所有编译目标可以通过以下命令查询.

```sh
rustc --print target-list
```

每一行输出唯一的标识了某一种二进制格式.这一字符串有一定的规律,也有一定的历史和人为因素.大致格式为3段式:`CPU架构-制造商-操作系统`,有的目标为4段式,最后一段是调用abi(如arm使用的`eabi`为嵌入式`abi`,`eabihf`表示支持硬件半浮点)或者使用的c语言库(如`gnu`表示使用`glibc`库,`musl`和`uclibc`分别表示使用对应的c语言库,`android`表示使用Android的`Bionic` c语言库,`msvc`表示Windows平台的`msvcrt`库).

显然适合编译操作系统的目标一般不在列表之中.一般来讲,为pc开发的系统内核使用的目标标识符为`x86_64-unknown-none`.也可以命名为`x86_64-unknown-myos`.

对于自定义目标,`rustc`不仅需要自定义的目标标识符,更重要的是需要llvm的目标配置(`target spec`)才能为非内置的目标编译.目标配置使用json定义,可以使用以下命令打印目标配置(目前需要nightly版的`rustc`)

```sh
rustc -Z unstable-options --print target-spec-json  # 当前平台的目标信息
rustc -Z unstable-options --target <内置的目标> --print target-spec-json  # 选择的目标信息
```

为内置目标交叉编译时,执行`cargo build --target <目标>`

为自定义目标交叉编译时,执行`cargo build --target <目标配置文件.json>`

以上命令很可能会失败:

```txt
error[E0463]: can't find crate for `core`
  |
  = note: the `x86_64-unknown-redox` target may not be installed
```

对于`tire 1`和`tire 2`平台,rust官方提供了二进制的`std`库文件,只需要使用rustup添加即可:

```sh
rustup target list  # 列出所有官方提供`std`库的二进制文件的目标,可见比rustc实际支持的编译目标少
rustup target add <目标>
```

对于`tire3`目标或自定义目标,官方不提供`core`的二进制文件.编译自己的库时,必须同时编译`core`和`compiler_builtins`两个只有源代码的库.

这些内置库目前不能由cargo自动编译,可以使用第三方工具`cargo-xbuild`来编译.

有关如何编写目标配置文件并编译的示例,见[blog os第二篇文章](https://os.phil-opp.com/minimal-rust-kernel/),[(中文翻译)](https://zhuanlan.zhihu.com/p/56433770)
