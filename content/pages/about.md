+++
title = "关于我"
path = "about"
template = "about.html"
+++

# 韩朴宇

- 开源爱好者
- Rust编程语言使用者/开发者/贡献者
- 熟练使用 Linux,Windows,macOS,FreeBSD 等操作系统，对操作系统的结构和本质有深入的理解。

## 联系方式/社交账号

邮箱: w12101111@outlook.com

博客: [12101111.github.io](https://12101111.github.io)

Github: [github.com/12101111/](https://github.com/12101111/)

知乎: [韩朴宇](https://www.zhihu.com/people/han-pu-yu/answers)

## 教育经历

本科: 哈尔滨工业大学 计算机科学与技术专业 计算机科学与工程方向

## 开源项目

### 1.[Gentoo overlay](https://github.com/12101111/overlay)

维护了数十款软件的Gentoo ebuild, 并包括了大量的补丁.

包含了将Electron移植到musl libc平台的所有补丁,并可以从源码编译Electron(6,7,8,9)和vscode.

### 2. [Net test](https://github.com/12101111/nettest)

全自动全功能的网络测试工具,可以测试网络延迟(TCP/UDP/ICMP)和带宽(TCP/QUIC),TCP带宽测试
兼容Speedtest服务器,本项目也提供了开源的Speedtest服务器替代.

### 3. [LLVM cross-compiled Linux From Scratch](https://12101111.github.io/llvm-cross-compile/)

使用纯LLVM项目下的工具编译Linux内核和用户态工具.只需要安装一份LLVM,Clang,LLD, 便可以为
LLVM支持的所有架构(交叉)编译出可运行的内核和操作系统.交叉编译不再需要费时费力编译gcc,只需要
按照该文章使用LLVM工具链即可.

使用此方法产生的Gentoo系统已经作为我的主力系统稳定的使用了半年,证明该方法本地可靠.

### 4.[操作系统](https://github.com/12101111/os)

一个使用UEFI启动,使用Rust编写的"操作系统".

子项目:

[bootuefi](https://github.com/12101111/bootuefi): 用于在`qemu`中运行和测试Rust编写的UEFI项目

[fbterm](https://github.com/12101111/fbterm): 建立于任何framebuffer上的终端模拟器,包含对vt100/ANSI X3.64序列的支持。

[uart_16550](https://github.com/12101111/uart_16550): x86平台的16550串口芯片的驱动程序库.

### 5.[Linux 0.11](https://github.com/12101111/Linux-0.11)

改造Linux0.11的`makefile`及部分源码,将Linux 0.11移植到现代编译器(GCC/Clang),并可以在Linux,Windows,macOS和Android上编译,运行,调试.

### 6.[rcc](https://github.com/12101111/rcc)

C 语言的“编译器”,使用Rust过程宏进行编译时运算,输入语法近似函数声明的产生式,在编译时生成LR(1)分析表,分析器和语法制导翻译程序.

### 7.[NES模拟器](https://github.com/12101111/oxidenes)

Rust编写的NES模拟器,已通过`nestest.nes`的所有CPU测试项目,支持的显示库/运行平台: `SDL2`/ UEFI Application / WebAssembly

子项目:[6502解释器/反汇编器](https://github.com/12101111/6502)

## 其他开源项目贡献

### [Rust语言](https://github.com/rust-lang/rust): Rust语言的编译期和标准库项目.

修复了Rust编译期的诊断输出bug,为rustc在musl host平台上编译proc macro添加了一个workaround。贡献量排名第757名(总共3160名贡献者)。

### [OpenZFS](https://github.com/openzfs/zfs): Solaris上卓越的Z文件系统在Linux和FreeBSD上的移植.

修复了若干使用Clang编译内核时的问题。

### [Gentoo-zh](https://github.com/microcai/gentoo-zh/): 中文用户的Gentoo overlay

添加并维护一些软件包，如腾讯QQ Linux版和fcitx5输入法。

### [Termux x11](https://github.com/termux/x11-packages): Android手机上的Linux发行版

添加bochs等模拟器软件，在手机上也可以运行Linux0.11等PC系统了。

## 校园生活

2017-2018学年担任"哈尔滨工业大学电脑110俱乐部"副主席职务.

2018-2019学年担任"哈尔滨工业大学电脑110俱乐部"主席职务.

2017年,2018年带领社团连续两年获得"哈尔滨工业大学十佳社团"称号.

大学期间参加社团志愿服务,累计为师生服务数百小时.
