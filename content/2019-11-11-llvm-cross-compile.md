+++
title = "LLVM cross-compiled Linux From Scratch: C & C++ libraries"
date = 2021-11-09
[taxonomies]
categories = ["Linux"]
tags = ["Linux","LLVM"]
+++

本系列文章将介绍如何使用LLVM工具链组装一个可用的Linux发行版. 本文已经更新以适应LLVM 13发生的一些变化.

## 面向读者

本文假定读者可以熟练的使用Unix-like系统,具有一定的C/C++编程能力,具有编译开源软件的经验.

## 什么是工具链

工具链,即一套具有工作顺序,可以编译出软件的工具.某些教科书上讲的"编译->汇编->链接"这一过程就粗略的描述了工具链的工作过程.

广义上讲,工具链除了编译期,汇编器,链接器还包含一些所有程序都会使用的库.

GNU工具链和LLVM工具链的对比:

| 项目               | GNU工具链           | LLVM工具链           |
| ------------------ | ------------------- | -------------------- |
| C 编译器           | `gcc`               | `clang`              |
| C++编译器          | `g++`               | `clang++`            |
| binutils           | GNU binutils        | LLVM binutils        |
| 汇编器             | GNU `as`            | 集成汇编器           |
| 链接器             | `ld.bfd`, `ld.gold` | LLVM linker `ld.lld` |
| 运行时(intrinsics) | `libgcc`            | `compiler-rt`        |
| 原子操作           | `libatomic`         | `compiler-rt`        |
| C 语言库           | GNU libc `glibc`    | LLVM libc            |
| C++ 标准库         | `libstdc++`         | `libc++`             |
| C++ ABI            | `Libsupcxx`         | `libc++abi`          |
| 栈展开(unwind)     | `libgcc_s`          | LLVM `libunwind`     |

<!-- more -->

1. [LLVM libc](https://llvm.org/docs/Proposals/LLVMLibC.html)由Google发起,仍处于早期阶段,gcc和clang皆可使用glibc,musl,BSD libc等开源C语言库,或者Apple或Windows提供的闭源C语言库.
2. LLVM项目的汇编器命令行前端集成在Clang中,使用`clang -c`命令行来使用,`llvm-as`用于编译LLVM IR,而不是各个平台的汇编.clang集成的汇编器声称与GNU as语法相互兼容,但是在x86之外的架构仍有不兼容之处.
3. 编译器运行时库提供了内建函数(intrinsics或builtins),这些函数提供复杂算数运算的实现.这些运算可能不能转换为单一汇编指令,而是翻译为一段C或汇编函数.
4. 有三种栈展开库都以`libunwind`命名,分别由LLVM,nongnu.org和PathScale开发,它们与`libgcc_s`都能提供 Itanium C++ ABI要求的`_Unwind_*`系列函数.libgcc_s.so的功能同时包含builtin和libunwind.
5. 更多详情见LLVM文档:[工具链](https://clang.llvm.org/docs/Toolchain.html)

由于各个库的API或ABI并不相同,因此闭源软件/预编译软件很大可能不能在LLVM工具链产生的系统上工作.例如:

1.Nvidia专有驱动的用户态组件.
2.QQ Linux版等闭源软件.

由于部分开源软件使用了GNU的C语言非标准扩展,因此暂时无法使用LLVM/musl编译.例如Chromium和Electron等软件需要十来个补丁才能编译通过.(本人已经成功将electron移植到musl平台下)

## 为什么使用LLVM

通常一份GNU工具链只能针对一个目标进行编译,但是LLVM是天生的交叉编译器,一份LLVM工具链可以同时为不同的目标编译.

要验证这一点,可以查看`llc --version`的输出,以下是我的结果:

```txt
> llc --version
LLVM (http://llvm.org/):
  LLVM version 13.0.0libcxx
  Optimized build.
  Default target: x86_64-gentoo-linux-musl
  Host CPU: skylake

  Registered Targets:
    aarch64    - AArch64 (little endian)
    aarch64_32 - AArch64 (little endian ILP32)
    aarch64_be - AArch64 (big endian)
    amdgcn     - AMD GCN GPUs
    arm        - ARM
    arm64      - ARM64 (little endian)
    arm64_32   - ARM64 (little endian ILP32)
    armeb      - ARM (big endian)
    avr        - Atmel AVR Microcontroller
    bpf        - BPF (host endian)
    bpfeb      - BPF (big endian)
    bpfel      - BPF (little endian)
    hexagon    - Hexagon
    lanai      - Lanai
    mips       - MIPS (32-bit big endian)
    mips64     - MIPS (64-bit big endian)
    mips64el   - MIPS (64-bit little endian)
    mipsel     - MIPS (32-bit little endian)
    msp430     - MSP430 [experimental]
    nvptx      - NVIDIA PTX 32-bit
    nvptx64    - NVIDIA PTX 64-bit
    ppc32      - PowerPC 32
    ppc32le    - PowerPC 32 LE
    ppc64      - PowerPC 64
    ppc64le    - PowerPC 64 LE
    r600       - AMD GPUs HD2XXX-HD6XXX
    riscv32    - 32-bit RISC-V
    riscv64    - 64-bit RISC-V
    sparc      - Sparc
    sparcel    - Sparc LE
    sparcv9    - Sparc V9
    systemz    - SystemZ
    thumb      - Thumb
    thumbeb    - Thumb (big endian)
    wasm32     - WebAssembly 32-bit
    wasm64     - WebAssembly 64-bit
    x86        - 32-bit X86: Pentium-Pro and above
    x86-64     - 64-bit X86: EM64T and AMD64
    xcore      - XCore
```

这意味着这份LLVM工具链可以为上面列出来的架构编译.

而对于GCC来说,GCC的源代码虽然支持许多平台,但是编译完成的GCC却只支持一个目标.同时自举GCC交叉编译器要多次编译GCC,及其耗时.

而我们只要安装一份LLVM/Clang,就可以同时为多个目标编译.

## 安装LLVM/Clang

本文需要主机上已经安装了一份现代的Linux发行版,以及*必要*的开发工具(如ArchLinux的`base-devel`),并且安装了Clang,LLVM,Compiler-rt和lld, 建议版本号大于10.0,本文以13.0.0为例.

本文在Archlinux,Alpine Linux,Gentoo Linux和Termux下测试.

本文不支持在Debian/Ubuntu/CentOS等发行版上运行,因为其打包的LLVM许多路径和程序名需要进行调整.建议使用[Alpine](https://wiki.alpinelinux.org/wiki/Alpine_Linux_in_a_chroot)进行操作.

## Example: 安装Alpine Linux chroot环境

本节介绍如何使用chroot安装一个Alpine环境. 也可以使用[systemd-nspawn](https://wiki.archlinux.org/title/systemd-nspawn), docker, lxc等容器工具设置此类环境, 但chroot终究是最通用且传统的方法.

以root用户执行以下命令

```output
export CHROOT=/var/chroot/alpine
wget -qO- http://mirrors.tuna.tsinghua.edu.cn/alpine/edge/main/x86_64/apk-tools-static-2.12.7-r3.apk | tar xvzC /tmp
mkdir -pv $CHROOT
/tmp/sbin/apk.static -X http://mirrors.tuna.tsinghua.edu.cn/alpine/edge/main -U --allow-untrusted --root $CHROOT --initdb add alpine-base
echo "http://mirrors.tuna.tsinghua.edu.cn/alpine/edge/main" >> $CHROOT/etc/apk/repositories
echo "http://mirrors.tuna.tsinghua.edu.cn/alpine/edge/community/" >> $CHROOT/etc/apk/repositories
```

你可以根据自身情况设置`CHROOT`变量,决定该chroot环境的安装地址.

注意开头的`apk-tools-static`的链接可能因版本号变动而失效, 可以通过查看[清华大学开源软件镜像站 Alpine Linux edge 文件列表页面](https://mirrors.tuna.tsinghua.edu.cn/alpine/edge/main/x86_64/)找到该工具的最新版链接.

挂载虚拟文件系统.每次重启系统后都需要运行一次.

```sh
mount /dev /var/chroot/alpine/dev --bind
mount -o remount,ro,bind /var/chroot/alpine/dev
mount -t proc none /var/chroot/alpine/proc
mount -o bind /sys /var/chroot/alpine/sys
cp /etc/resolv.conf /var/chroot/alpine/etc/
```

进入chroot环境

```shell
chroot /var/chroot/alpine/ /bin/ash -l
```

安装需要的软件包. 请注意alpine并没有更新到llvm13, 因此下文所有的llvm13都应该被替换为llvm12.

```shell
apk update
apk add make cmake ninja llvm12 llvm12-dev llvm12-static clang lld gcc musl-dev rsync python3 ncurses-dev flex bison perl linux-headers libressl-dev elfutils-dev autoconf automake libtool lz4 compiler-rt compiler-rt-static
```

这些软件分别有以下作用:

1. GNU make: Linux,musl以及其他软件的构建工具
2. cmake: LLVM使用cmake作为构建工具
3. ninja: ninja与cmake配合可以加快构建速度.samurai是兼容ninja的工具,ninja使用C++编写,而samurai使用C编写.
4. gcc,musl-dev: 编译主机(Host)程序.
5. rsync: 安装Linux头文件
6. python3: LLVM构建脚本
7. ncurses-dev: Linux的`menuconfig`/`nconfig`配置菜单的依赖.
8. flex,bison,perl,linux-headers,libressl-dev,elfutils-dev: Linux的编译时依赖
9. autoconf automake libtool: autotools工具,用于重新生成configure脚本
10. lz4: 压缩initramfs

## 环境变量与工作文件夹

首先选择编译目标三元组(target triple),格式:CPU架构-制造商(可忽略或为unknown)-操作系统-libc库

如果想编译在PC上运行的系统,使用`x86_64-linux-musl`.如果想在ARM开发板或者手机(chroot)上运行,使用`aarch64-linux-musl`.armv7的Soc应该选择`armv7-linux-musleabihf`.其他架构请自行查询合适的目标.

在shell中执行以下内容

```bash
#替换为要使用的目标
export TARGET=aarch64-linux-musl
export CROSS_COMPILE=${TARGET}-
export SYSROOT=$HOME/${TARGET}/sysroot
export INITRAMFS=$HOME/${TARGET}/initramfs
export DISTDIR=$HOME/${TARGET}/build
```

建立工作文件夹.`sysroot`用于存放自举需要使用的库文件和头文件.`initramfs`用于存放initramfs的根文件系统.`distdir`用于存放下载的源代码和编译的中间产物.

```shell
mkdir -pv $SYSROOT
mkdir -pv $INITRAMFS
mkdir -pv $DISTDIR
```

根据传统习惯建立工具链的符号链接.

```shell
export XDG_BIN=$HOME/.local/bin
mkdir -pv $XDG_BIN
ln -s `which lld` $XDG_BIN/${CROSS_COMPILE}ld
ln -s `which lld` $XDG_BIN/${CROSS_COMPILE}ld.lld
ln -s `which clang` $XDG_BIN/${CROSS_COMPILE}gcc
ln -s `which clang` $XDG_BIN/${CROSS_COMPILE}clang
ln -s `which clang++` $XDG_BIN/${CROSS_COMPILE}g++
ln -s `which clang++` $XDG_BIN/${CROSS_COMPILE}clang++
for i in ar nm objcopy objdump ranlib strip;do
  ln -s `which llvm-$i` $XDG_BIN/${CROSS_COMPILE}$i
done
export PATH="$XDG_BIN:$PATH"
```

将以下内容保存到`$HOME/${TARGET}/env`中.

`COMMON_FLAGS`需要根据目标CPU进行调整, `-tune=cortex-a76`表示针对skylake微架构优化,可用的值可以运行`$ARCH-linux-musl-clang --print-supported-cpus`查看.对于arm平台,如果内存较小,可以使用`Os`而不是`O2`.

```bash
#替换为要使用的目标
export TARGET=aarch64-linux-musl
#或者
export TARGET=x86_64-linux-musl

export CROSS_COMPILE=${TARGET}-
export SYSROOT=$HOME/${TARGET}/sysroot
export INITRAMFS=$HOME/${TARGET}/initramfs
export DISTDIR=$HOME/${TARGET}/build

export COMMON_FLAGS="-mcpu=cortex-a76 -O2 -pipe --sysroot=${SYSROOT}"
#或者
export COMMON_FLAGS="-march=skylake -mtune=skylake -O2 -pipe --sysroot=${SYSROOT}"

export CFLAGS="${COMMON_FLAGS}"
export CXXFLAGS="${COMMON_FLAGS} -stdlib=libc++"
export LDFLAGS="-fuse-ld=lld -rtlib=compiler-rt -flto=thin"
export XDG_BIN=$HOME/.local/bin
export PATH="$XDG_BIN:$PATH"
```

导入这些环境变量到当前的shell

```shell
source $HOME/aarch64-linux-musl/env
```

## 安装Linux头文件

首先下载Linux源代码.

```shell
cd $DISTDIR && wget -qO- https://mirrors.tuna.tsinghua.edu.cn/kernel/v5.x/linux-5.15.1.tar.xz | tar xvJ && mv linux-5.15.1 linux && cd linux
```

对于树莓派,从Github获取Linux源码.

```shell
cd $DISTDIR && git clone --depth=1 --branch rpi-5.15.y https://github.com/raspberrypi/linux && cd linux
```

其他开发板根据情况选择主线内核或BSP内核。

然后安装头文件:

```shell
make ARCH=arm64 INSTALL_HDR_PATH=$SYSROOT headers_install #这一步需要rsync
```

## 安装musl libc的头文件

```shell
cd $DISTDIR && wget -qO- https://musl.libc.org/releases/musl-1.2.2.tar.gz | tar xvz && cd musl-1.2.2
./configure --prefix=/
DESTDIR=$SYSROOT make install-headers
```

## 交叉编译Compiler-RT builtins

如果你没有在交叉编译,则不需要这一步骤.请确认存在`/usr/lib/clang/13.0.0/lib/linux/libclang_rt.builtins-*.a`这一文件(clang版本号可能有所不同).例如`libclang_rt.builtins-x86_64.a`或`libclang_rt.builtins-aarch64.a`

如果没有目标平台的文件,则需要编译`compiler-rt`.下面以aarch64架构为例.

首先下载并解压其源代码.

```shell
cd $DISTDIR && wget -qO- https://github.com/llvm/llvm-project/releases/download/llvmorg-13.0.0/llvm-project-13.0.0.src.tar.xz | tar xvJ && cd llvm-project-13.0.0.src/compiler-rt
```

开始编译:

```shell
# 按照cmake的习惯,建立build文件夹
mkdir build && cd build
# 生成ninja编译文件
cmake ../ -G Ninja \
-DCOMPILER_RT_BUILD_BUILTINS=ON \
-DCOMPILER_RT_INCLUDE_TESTS=OFF \
-DCOMPILER_RT_BUILD_CRT=ON \
-DCOMPILER_RT_BUILD_SANITIZERS=OFF \
-DCOMPILER_RT_BUILD_XRAY=OFF \
-DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
-DCOMPILER_RT_BUILD_PROFILE=OFF \
-DCOMPILER_RT_BUILD_MEMPROF=OFF \
-DCOMPILER_RT_BUILD_ORC=OFF \
-DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
-DCMAKE_ASM_COMPILER=clang \
-DCMAKE_C_COMPILER=clang \
-DCMAKE_CXX_COMPILER=clang++ \
-DCMAKE_ASM_COMPILER_TARGET=$TARGET \
-DCMAKE_C_COMPILER_TARGET=$TARGET \
-DCMAKE_CXX_COMPILER_TARGET=$TARGET \
-DCMAKE_SYSROOT=$SYSROOT \
-DCMAKE_INSTALL_PREFIX="/usr/lib/clang/13.0.0/" \
-DCMAKE_C_COMPILER_WORKS=1 \
-DCMAKE_CXX_COMPILER_WORKS=1 \
-DCMAKE_SIZEOF_VOID_P=8
# 编译
ninja
# 会安装3个文件libclang_rt.builtins-aarch64.a, clang_rt.crtbegin-aarch64.o, clang_rt.crtend-aarch64.o
sudo ninja install
```

## 编译musl libc

```shell
cd $DISTDIR/musl-1.2.2
make distclean
./configure --prefix=/ LIBCC=/usr/lib/clang/13.0.0/lib/linux/libclang_rt.builtins-aarch64.a #手动指定上一步编译的文件
make -j13 # CPU线程数+1
DESTDIR=$SYSROOT make install
```

## 编译LLVM libunwind

```shell
cd $DISTDIR/llvm-project-13.0.0.src/runtimes && mkdir libunwind-build && cd libunwind-build

cmake ../ -G Ninja \
-DLLVM_ENABLE_RUNTIMES="libunwind" \
-DLIBUNWIND_USE_COMPILER_RT=ON \
-DLIBUNWIND_SUPPORTS_FNO_EXCEPTIONS_FLAG=1 \
-DCMAKE_ASM_COMPILER=clang \
-DCMAKE_C_COMPILER=clang \
-DCMAKE_CXX_COMPILER=clang++ \
-DCMAKE_ASM_COMPILER_TARGET=$TARGET \
-DCMAKE_C_COMPILER_TARGET=$TARGET \
-DCMAKE_CXX_COMPILER_TARGET=$TARGET \
-DCMAKE_C_FLAGS="$CFLAGS" \
-DCMAKE_CXX_FLAGS="$CXXFLAGS" \
-DCMAKE_ASM_FLAGS="$CFLAGS" \
-DCMAKE_SHARED_LINKER_FLAGS="$LDFLAGS -unwindlib=none" \
-DCMAKE_SYSROOT=$SYSROOT \
-DCMAKE_INSTALL_PREFIX=$SYSROOT \
-DCMAKE_C_COMPILER_WORKS=1 \
-DCMAKE_CXX_COMPILER_WORKS=1

ninja
ninja install

cp $DISTDIR/llvm-project-13.0.0.src/libunwind/include/*.h $SYSROOT/include
```

本来libunwind是可以和libc++以及libc++abi一同编译的(`LLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind"`,但是由于一些小问题, `libunwind.so.1`在链接时`clang`会自作聪明插入一个`--as-needed -l:libunwind.so`, 然而我们正在编译的就是libunwind本身,所以必须在命令行中加上`-unwindlib=none`来阻止这个行为. 具体源码见[这里](https://github.com/llvm/llvm-project/blob/d7b669b3a30345cfcdb2fde2af6f48aa4b94845d/clang/lib/Driver/ToolChains/CommonArgs.cpp#L1461), 只要用了`-rtlib=compiler-rt`选项, clang就会自动链接libunwind.

## 编译libc++和libc++abi

```shell
cd $DISTDIR/llvm-project-13.0.0.src/runtimes && mkdir build && cd build

cmake ../ -G Ninja \
-DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi" \
-DLIBCXXABI_USE_LLVM_UNWINDER=YES \
-DLIBCXX_HAS_MUSL_LIBC=ON \
-DCMAKE_C_COMPILER=clang \
-DCMAKE_CXX_COMPILER=clang++ \
-DCMAKE_C_COMPILER_TARGET=$TARGET \
-DCMAKE_CXX_COMPILER_TARGET=$TARGET \
-DCMAKE_C_FLAGS="$CFLAGS" \
-DCMAKE_CXX_FLAGS="$CXXFLAGS" \
-DCMAKE_SYSROOT=$SYSROOT \
-DCMAKE_INSTALL_PREFIX=$SYSROOT \
-DCMAKE_CXX_COMPILER_WORKS=1

ninja
ninja install
```

## 结束

到目前为止,我们已经编译了compiler-rt, musl libc, libc++, libc++abi, libunwind, 已经可以使用此sysroot编译任何C和C++的库和程序了.

接下来我们要使用LLVM工具链编译其他工具及内核,以组装出一个精简的Linux操作系统.
