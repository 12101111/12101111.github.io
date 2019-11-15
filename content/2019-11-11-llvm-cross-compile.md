+++
title = "LLVM cross-compiled Linux From Scratch (1)"
date = 2019-11-14
[taxonomies]
categories = ["Linux"]
tags = ["Linux","LLVM"]
+++

通常一份GNU工具链只能针对一个目标进行编译,但是LLVM是天生的交叉编译器,一份LLVM工具链可以同时为不同的目标编译.

要验证这一点,可以查看`llc --version`的输出,以下是我的结果:

```txt
LLVM (http://llvm.org/):
  LLVM version 9.0.0libcxx
  Optimized build.
  Default target: x86_64-gentoo-linux-musl
  Host CPU: skylake

  Registered Targets:
    aarch64    - AArch64 (little endian)
    aarch64_32 - AArch64 (little endian ILP32)
    aarch64_be - AArch64 (big endian)
    arm        - ARM
    arm64      - ARM64 (little endian)
    arm64_32   - ARM64 (little endian ILP32)
    armeb      - ARM (big endian)
    riscv32    - 32-bit RISC-V
    riscv64    - 64-bit RISC-V
    thumb      - Thumb
    thumbeb    - Thumb (big endian)
    wasm32     - WebAssembly 32-bit
    wasm64     - WebAssembly 64-bit
    x86        - 32-bit X86: Pentium-Pro and above
    x86-64     - 64-bit X86: EM64T and AMD64
```

这意味着这份LLVM工具链可以为上面列出来的架构编译.

本文将用LLVM工具链从零交叉编译出目标架构为`aarch64-linux-musl`的Linux操作系统.

<!-- more -->

本文需要主机上已经安装了一份现代的Linux发行版,以及*必要*的开发工具(如Debian的`build-essential`或ArchLinux的`base-devel`),并且安装了Clang 9,LLVM 9和lld 9.

某些需要的工具为:`bash bzip2 coreutils diffutils findutils gawk grep gzip m4 make ncurses patch sed tar cmake ninja rsync`.这个列表可能不完整,其中某些工具可以被busybox代替,但我没有测试过.

如上文所示,我使用的发行版为Gentoo Linux,并且大部分包通过LLVM工具链编译,但这与本文无关.

## GNU工具链和LLVM工具链的对比

| 项目               | GNU工具链           | LLVM工具链           |
| ------------------ | ------------------- | -------------------- |
| C 编译器           | `gcc`               | `clang`              |
| C++编译器          | `g++`               | `clang++`            |
| binutils           | GNU binutils        | LLVM binutils        |
| 汇编器             | GNU `as`            | 集成汇编器           |
| 链接器             | `ld.bfd`, `ld.gold` | LLVM linker `ld.lld` |
| 运行时(intrinsics) | `libgcc`            | `compiler-rt`        |
| 原子操作           | `libatomic`         | `compiler-rt`        |
| C 语言库           | GNU libc `glibc`    | `musl`               |
| C++ 标准库         | `libstdc++`         | `libc++`             |
| C++ ABI            | `Libsupcxx`         | `libc++abi`          |
| 栈展开(unwind)     | `libgcc_s`          | LLVM `libunwind`     |

1. LLVM项目暂时没有libc库,gcc和clang皆可使用glibc或musl等C语言库.
2. LLVM项目的汇编器命令行前端集成在Clang中,使用`clang -c`命令行来使用,`llvm-as`用于编译LLVM IR,而不是各个平台的汇编.clang集成的汇编器有时不能处理GNU汇编的语法,因此我们需要安装GNU as.
3. 编译器运行时库提供了内建函数(intrinsics或builtins),这些函数提供复杂算数运算的实现.这些运算可能不能转换为单一汇编指令,而是翻译为一段C或汇编函数.
4. 有三种栈展开库都以`libunwind`命名,分别由LLVM,nongnu.org和PathScale开发,它们与`libgcc_s`都能提供 Itanium C++ ABI要求的`_Unwind_*`系列函数.
5. 更多详情见LLVM文档:[工具链](https://clang.llvm.org/docs/Toolchain.html)

## 环境变量

```bash
# 交叉编译的目标(target triple),格式:CPU架构-制造商(可忽略或为unknown)-操作系统-libc库
export TARGET=aarch64-linux-musl
export SYSROOT=/usr/${TARGET}/sysroot
export ROOTFS=/usr/${TARGET}/rootfs
export CROSS_COMPILE=${TARGET}-
export CFLAGS="--sysroot=${SYSROOT}"
export CXXFLAGS="-stdlib=libc++ --sysroot=${SYSROOT}"
export LDFLAGS="-fuse-ld=lld -rtlib=compiler-rt"
```

请注意,此环境变量需要在root和普通用户都有效,因为部分命令需要root才能执行.

## 建立文件夹

```shell
mkdir -pv /usr/$TARGET/{sysroot,rootfs}
```

sysroot用于存放编译使用的库文件和头文件，rootfs存放编译完的程序和库文件.建议此目录保持root所有

## 使用符号链接建立交叉编译工具链

```shell
ln -s `which lld` /usr/local/bin/${CROSS_COMPILE}ld
ln -s `which lld` /usr/local/bin/${CROSS_COMPILE}ld.lld
ln -s `which clang` /usr/local/bin/${CROSS_COMPILE}gcc
ln -s `which clang` /usr/local/bin/${CROSS_COMPILE}clang
ln -s `which clang++` /usr/local/bin/${CROSS_COMPILE}g++
ln -s `which clang++` /usr/local/bin/${CROSS_COMPILE}clang++
for i in ar nm objcopy objdump ranlib strip;do
  ln -s `which llvm-$i` /usr/local/bin/${CROSS_COMPILE}$i
done
```

## 安装头文件

这一步需要获得Linux源代码,对于有主线支持的平台,可以从kernel.org获取Linux源码.

```shell
cd ~
git clone --depth=1 --branch v5.4-rc7 https://mirrors.tuna.tsinghua.edu.cn/git/linux.git && cd linux
make ARCH=arm64 headers_check
make ARCH=arm64 INSTALL_HDR_PATH=$SYSROOT headers_install
```

你会看到`which: no aarch64-linux-musl-elfedit in (/usr/lib/llvm/9/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/bin)`,这是正常现象,Linux的Makefile用这个llvm-binutils不存在的工具[寻找GNU工具链的位置](https://github.com/torvalds/linux/commit/ad15006cc78459d059af56729c4d9bed7c7fd860),但我们不需要GNU工具链.

## 安装musl libc的头文件

```shell
cd ~
curl -L https://www.musl-libc.org/releases/musl-1.1.24.tar.gz | tar xvz && cd musl-1.1.24
./configure --prefix=/
DESTDIR=$SYSROOT make install-headers
```

## 交叉编译Compiler-RT

首先要做的是编译一份arm平台的`compiler-rt`

首先下载并解压其源代码.

```shell
cd ~
curl -L http://releases.llvm.org/9.0.0/compiler-rt-9.0.0.src.tar.xz | tar xvJ && cd compiler-rt-9.0.0.src
```

我们需要修改一下`CMakeLists.txt`以跳过对C编译器的检测,手动指定指针长度,因为现在C编译器不能链接出一个可执行的程序.

对于32位的系统,`CMAKE_SIZEOF_VOID_P`为`4`,64位的系统则为`8`

```diff
--- CMakeLists.txt      2019-08-26 12:32:35.000000000 +0000
+++ CMakeLists.txt      2019-11-11 08:34:05.860000000 +0000
@@ -5,6 +5,10 @@

 cmake_minimum_required(VERSION 3.4.3)

+set(CMAKE_C_COMPILER_WORKS 1)
+set(CMAKE_CXX_COMPILER_WORKS 1)
+set(CMAKE_SIZEOF_VOID_P 8)
+
 if(POLICY CMP0075)
   cmake_policy(SET CMP0075 NEW)
 endif()
```

```shell
# 按照cmake的习惯,建立build文件夹
mkdir build && cd build
# 生成ninja编译文件,最后的CMAKE_INSTALL_PREFIX应为对应的clang安装到的文件夹,下面应该有lib/linux/clang_rt.crtbegin-*等
cmake ../ -G Ninja -DCOMPILER_RT_BUILD_BUILTINS=ON -DCOMPILER_RT_BUILD_SANITIZERS=OFF -DCOMPILER_RT_BUILD_XRAY=OFF -DCOMPILER_RT_BUILD_LIBFUZZER=OFF -DCOMPILER_RT_BUILD_PROFILE=OFF -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" -DCMAKE_C_COMPILER_TARGET=$TARGET -DCMAKE_ASM_COMPILER_TARGET=$TARGET -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON -DCMAKE_SYSROOT=$SYSROOT -DCMAKE_INSTALL_PREFIX="/usr/lib/clang/9.0.0/"
# 编译
ninja
# 会安装3个文件libclang_rt.builtins-aarch64.a, clang_rt.crtbegin-aarch64.o, clang_rt.crtend-aarch64.o
ninja install
```

## 交叉编译musl libc

```shell
cd ~/musl-1.1.24
make distclean
./configure --prefix=/
make -j7 # CPU线程数+1
DESTDIR=$SYSROOT make install
```

请注意configure脚本的输出中,`using compiler runtime libraries:`,此处应为上面安装的`libclang_rt.builtins-aarch64.a`,如果不是,则使用以下的configure命令.

```shell
./configure --prefix=/ LIBCC=/usr/lib/clang/9.0.0/lib/linux/libclang_rt.builtins-aarch64.a
```

## 交叉编译busybox

```shell
cd ~
curl -L https://busybox.net/downloads/busybox-1.31.1.tar.bz2 | tar xvj && cd busybox-1.31.1
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE menuconfig
# Settings -> Build static binary (no shared libs)
make ARCH=arm64 CFLAGS="-Wignored-optimization-argument ${CFLAGS}" CROSS_COMPILE=$CROSS_COMPILE -j7
mkdir $ROOTFS/bin && cp busybox $ROOTFS/bin
```

验证可以使用

```shell
$file busybox
busybox: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, stripped
$qemu-aarch64-static busybox uname -m
aarch64
```

## 交叉编译LLVM libunwind

```shell
cd ~
curl -L http://releases.llvm.org/9.0.0/libunwind-9.0.0.src.tar.xz | tar xvJ &&  cd libunwind-9.0.0.src
```

编辑`CMakeLists.txt`以跳过对C编译器的检测,否则会出错说The C++ compiler is not able to compile a simple test program,因为我们现在没有libc++.

只需要加入`set(CMAKE_CXX_COMPILER_WORKS 1)`即可.

再应用以下补丁,否则在一些系统上LLVM会提示libstdc++版本过低.实际上我们不使用libstdc++(GCC的C++标准库),而是使用LLVM的libc++.

```diff
--- CMakeLists.txt      2019-11-15 21:08:04.680000000 +0800
+++ CMakeLists.txt      2019-11-15 21:08:00.950000000 +0800
@@ -78,8 +78,8 @@
     # Enable warnings, otherwise -w gets added to the cflags by HandleLLVMOptions.
     set(LLVM_ENABLE_WARNINGS ON)
     list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_PATH}")
-    include("${LLVM_CMAKE_PATH}/AddLLVM.cmake")
-    include("${LLVM_CMAKE_PATH}/HandleLLVMOptions.cmake")
+    #include("${LLVM_CMAKE_PATH}/AddLLVM.cmake")
+    #include("${LLVM_CMAKE_PATH}/HandleLLVMOptions.cmake")
   else()
     message(WARNING "Not found: ${LLVM_CMAKE_PATH}")
   endif()
```

```shell
mkdir build && cd build
cmake ../ -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_CXX_COMPILER_TARGET=$TARGET -DCMAKE_C_COMPILER_TARGET=$TARGET -DCMAKE_SYSROOT=$SYSROOT -DCMAKE_INSTALL_PREFIX=$SYSROOT -DCMAKE_CXX_FLAGS="$CXXFLAGS"
ninja
ninja install
```

## 交叉编译libc++和libc++abi

同样的,编辑`CMakeLists.txt`,加入`set(CMAKE_CXX_COMPILER_WORKS 1)`.

```shell
cd ~
curl -L http://releases.llvm.org/9.0.0/libcxx-9.0.0.src.tar.xz | tar xvJ
curl -L http://releases.llvm.org/9.0.0/libcxxabi-9.0.0.src.tar.xz | tar xvJ && cd libcxxabi-9.0.0.src
mkdir build && cd build
cmake ../ -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_CXX_COMPILER_TARGET=$TARGET -DCMAKE_C_COMPILER_TARGET=$TARGET -DCMAKE_SYSROOT=$SYSROOT -DCMAKE_INSTALL_PREFIX=$SYSROOT -DLIBCXXABI_USE_COMPILER_RT=YES -DLIBCXXABI_USE_LLVM_UNWINDER=YES -DLIBCXXABI_LIBCXX_PATH="$HOME/libcxx-9.0.0.src"
ninja
ninja install
cd $HOME/libcxx-9.0.0.src && mkdir build && cd build
cmake ../ -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_CXX_COMPILER_TARGET=$TARGET -DCMAKE_C_COMPILER_TARGET=$TARGET -DCMAKE_SYSROOT=$SYSROOT -DCMAKE_INSTALL_PREFIX=$SYSROOT -DLIBCXX_CXX_ABI=libcxxabi -DLIBCXX_USE_COMPILER_RT=YES -DLIBCXX_HAS_MUSL_LIBC=ON -DLIBCXX_CXX_ABI_INCLUDE_PATHS="$HOME/libcxxabi-9.0.0.src/include"
ninja
ninja install
```

## Binutils(GNU as)

像Linux或uboot这样大量使用内联汇编的项目,Clang内置的汇编器有时会不认GNU汇编的语法,因此最好使用GNU的汇编器gas.

```shell
unset CFLAGS CXXFLAGS LDFLAGS
curl -L https://mirrors.tuna.tsinghua.edu.cn/gnu/binutils/binutils-2.33.1.tar.xz | tar xvJ && cd binutils-2.33.1
./configure --target=$TARGET --disable-nls --disable-werror --disable-ld --disable-gold --disable-gprof --disable-binutils
make -j7
make install
```

## 编译Linux

```shell
cd linux
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE menuconfig
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE Image modules
```

未完待续...
