+++
title = "为什么Linux不能运行FreeBSD的elf"
date = 2020-02-12
[taxonomies]
categories = ["Linux"]
tags = ["Linux","FreeBSD","ELF"]
+++
# Linux和FreeBSD都使用ELF,但是诸多差异导致二者二进制并不兼容

查看ELF信息可以使用两个程序,`file`和`readelf`
file 这个程序可以快速识别ELF文件的不同之处(也可以识别其他文件).readelf则可以读取大部分ELF文件的信息.

## C语言库不同导致的加载器和ABI差异

除了少数如busybox的程序,绝大多数程序都动态链接到C语言库.
不同的C语言库使用不同的加载器（interpreter），这是阻碍绝大多数elf文件在各个系统之间兼容的主要因素，甚至同为Linux,glibc和musl也互不兼容。

这是静态链接的Linux下的busybox (ARM64)
bin/busybox: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, stripped
这是动态链接到GNU libc的Linux下的bash
/bin/bash: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, stripped
这是动态链接到musl libc的Linux下的zfs
zfs: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-x86_64.so.1, stripped
这是FreeBSD的zfs
sbin/zfs: ELF 64-bit LSB executable, x86-64, version 1 (FreeBSD), dynamically linked, interpreter /libexec/ld-elf.so.1, for FreeBSD 12.1 (1201511), FreeBSD-style, stripped
首先最明显的就是interpreter不同,当然根本原因还是C语言库不同.

使用`readelf -l`可以查看程序头表(Program header table),interpreter的位置就在这里.

```output
$ readelf -l sbin/zfs

Elf 文件类型为 EXEC (可执行文件)
Entry point 0x20c000
There are 11 program headers, starting at offset 64

程序头：
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000200040 0x0000000000200040
                 0x0000000000000268 0x0000000000000268  R      0x8
  INTERP         0x00000000000002a8 0x00000000002002a8 0x00000000002002a8
                 0x0000000000000015 0x0000000000000015  R      0x1
      [Requesting program interpreter: /libexec/ld-elf.so.1]
  LOAD           0x0000000000000000 0x0000000000200000 0x0000000000200000
                 0x000000000000be84 0x000000000000be84  R      0x1000
  LOAD           0x000000000000c000 0x000000000020c000 0x000000000020c000
                 0x000000000000cb50 0x000000000000cb50  R E    0x1000
  LOAD           0x0000000000019000 0x0000000000219000 0x0000000000219000
                 0x00000000000001e0 0x00000000000001e0  RW     0x1000
  LOAD           0x000000000001a000 0x000000000021a000 0x000000000021a000
                 0x0000000000000c80 0x00000000000015b8  RW     0x1000
  DYNAMIC        0x0000000000019028 0x0000000000219028 0x0000000000219028
                 0x00000000000001b0 0x00000000000001b0  RW     0x8
  GNU_RELRO      0x0000000000019000 0x0000000000219000 0x0000000000219000
                 0x00000000000001e0 0x00000000000001e0  R      0x1
  GNU_EH_FRAME   0x000000000000ad48 0x000000000020ad48 0x000000000020ad48
                 0x00000000000002b4 0x00000000000002b4  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x0
  NOTE           0x00000000000002c0 0x00000000002002c0 0x00000000002002c0
                 0x0000000000000030 0x0000000000000030  R      0x4

 Section to Segment mapping:
  段节...
   00
   01     .interp
   02     .interp .note.tag .dynsym .gnu.version .gnu.version_r .gnu.hash .hash .dynstr .rela.dyn .rela.plt .rodata .eh_frame_hdr .eh_frame
   03     .text .init .fini .plt
   04     .ctors .dtors .jcr .dynamic .got
   05     .data .got.plt .bss
   06     .dynamic
   07     .ctors .dtors .jcr .dynamic .got
   08     .eh_frame_hdr
   09
   10     .note.tag
```

在Linux下运行FreeBSD的程序就会提示`zsh: 没有那个文件或目录: sbin/zfs`,实际上是interpreter找不到的通用提示.(当然也很具有迷惑性,因为文件明明就在那里),

当然把interpreter放过去也没有用,直接显示`[1]    18145 segmentation fault  sbin/zfs`, 可见Linux并不检查其他的东西就直接运行这个elf可执行文件,结果就是直接段错误. dmesg有如下日志:

```
[ 9734.919582] ld-elf.so.1[12587]: segfault at 0 ip 00007f28598cda3f sp 00007fffce706b38 error 4 in ld-elf.so.1[7f28598cd000+1a000]
[ 9734.919594] Code: 57 08 48 89 c7 5d e9 c0 58 00 00 55 48 89 e5 41 57 41 56 41 55 41 54 53 48 81 ec b8 0a 00 00 49 89 d4 48 8d 47 08 48 89 45 b8 <48> 8b 1f 48 89 d8 48 c1 e0 20 48 b9 00 00 00 00 01 00 00 00 48 01
```

## 骚操作

通常不同的interpreter下的库不能相互加载,但是实际上是可以通过一些兼容层(shim)使他们共存.
[AdelieLinux/gcompat](https://github.com/AdelieLinux/gcompat) 这个项目实现了一个musl下的glibc的加载器,使得musl可以执行部分glibc的程序(通过补全符号的方式),但是实际使用上,由于不同libc之间不只是符号不同,ABI也不同,比如一些结构体的大小不一样,这样直接兼容是不太可能的.我实际测试来看,gcompat能在musl系统上执行nvidia驱动的`nvidia-smi`命令,但是启动Xorg会直接段错误.

FreeBSD这边也有类似的骚操作: https://github.com/shkhln/nvshim 用Linux兼容层下的Nvidia驱动使得FreeBSD的程序用上Nvidia的Vulkan
这里插播一句fxxk nvidia.总之这种操作实用价值不高.

## ELF的ABI字段

BSD更加接近Unix©,因此比Linux要严谨一些,至少会检查ABI的类型.这个信息位于ELF头的0x7位置,可以用`readelf -h`查看:

```
readelf -h zfs
ELF 头：
  Magic：  7f 45 4c 46 02 01 01 09 00 00 00 00 00 00 00 00
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - FreeBSD
  ABI 版本:                          0
  类型:                              EXEC (可执行文件)
  系统架构:                          Advanced Micro Devices X86-64
  版本:                              0x1
  入口点地址：              0x20c000
  程序头起点：              64 (bytes into file)
  Start of section headers:          110648 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         11
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
$ readelf -h /sbin/zfs
ELF 头：
  Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              EXEC (可执行文件)
  系统架构:                          Advanced Micro Devices X86-64
  版本:                              0x1
  入口点地址：              0x20e000
  程序头起点：              64 (bytes into file)
  Start of section headers:          126648 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         11
  Size of section headers:           64 (bytes)
  Number of section headers:         28
  Section header string table index: 27
```

Linux的ABI是默认的0号(System V), FreeBSD使用0x9,OpenBSD使用0xC,NetBSD使用0x2.但实际上0x3是分给Linux的,由于GNU的[一些混乱的操作](https://sourceware.org/bugzilla/show_bug.cgi?id=12913)而没有用上.

FreeBSD则靠这个来检查是否应该用linux兼容层来执行elf文件,还有一个程序[brandelf](https://www.freebsd.org/cgi/man.cgi?query=brandelf&sektion=1&manpath=freebsd-release-ports)来修改ABI信息.

## syscall ABI不兼容

更深层次的讲,Linux和FreeBSD内核的系统调用也互不兼容.Linux提供稳定的syscall ABI, FreeBSD在每一个大版本中保证内核和用户库的ABI都不变,但是不在版本间做出保证.FreeBSD的Linux兼容层内核模块就需要把Linux的系统调用翻译到FreeBSD上(非常类似windows10的WSL1).

FreeBSD是开源的,Linux兼容层代码并不复杂,就是大量的转换代码:

https://github.com/freebsd/freebsd/tree/master/sys/compat/linux

