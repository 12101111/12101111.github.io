+++
title = "LLVM cross-compiled Linux From Scratch: Bootable kernel and initramfs"
date = 2020-01-23
[taxonomies]
categories = ["Linux"]
tags = ["Linux","LLVM"]
+++

本文章是LLVM编译Linux系统的第二篇文章,介绍如何使用LLVM/Clang/musl工具链编译一个可以启动的Linux内核.

<!-- more -->

## 编译GNU assembler

LLVM提供了大部分binutils的工具,但是LLVM的内置汇编器并不能编译一些Linux的汇编,因此需要编译`gas`

```shell
cd $DISTDIR && wget -qO- https://mirrors.tuna.tsinghua.edu.cn/gnu/binutils/binutils-2.33.1.tar.xz | tar xvJ && cd binutils-2.33.1
unset CFLAGS CXXFLAGS LDFLAGS #这部分是为Host编译的
./configure --prefix= --target=$TARGET --disable-nls --disable-werror --disable-ld --disable-gold --disable-gprof --disable-binutils
make -j7
DESTDIR=$HOME/.local/ make install
```

## 编译Linux内核

打开设置菜单

```shell
export ARCH=arm64 # 对应内核arch目录的子目录名/
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE menuconfig
```

请自行根据硬件调整配置,可以参考[Gentoo的文档](https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide)

总的来说,可以通过启动livecd来判断所需要的内核组件.对于计算机启动时就会加载的模块,选择`[Y]`,对于USB设备则可以选择为`[M]`,其他用不到的选择为`[N]`.

对于x86_64 UEFI启动的计算机,建议编译时打开`CONFIG_EFI_STUB`选项,这样就可以直接从UEFI shell启动了.

对于树梅派3/4等arm设备,内核源码提供预先编写的配置.

```shell
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE bcmrpi3_defconfig # Pi 2 (1.2+), Pi 3, Pi 3+, or Compute Module 3
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE bcm2711_defconfig # Pi 4b+
```

如果只是想在虚拟机中测试的话，可以选择：

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE tinyconfig
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE kvmconfig
```

然后编译内核.

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -j7
```

对于x86系统会有以下输出

```output
  BUILD   arch/x86/boot/bzImage
Setup is 15916 bytes (padded to 16384 bytes).
System is 19892 kB
CRC 17bccae5
Kernel: arch/x86/boot/bzImage is ready  (#1)
```

可以在`arch/x86/boot/bzImage`找到编译好的内核.可以用UEFI shell或者grub引导器启动这个内核.

对于arm系统会有以下输出

```output
  LD      vmlinux
  SORTEX  vmlinux
  SYSMAP  System.map
  OBJCOPY arch/arm64/boot/Image
  GZIP    arch/arm64/boot/Image.gz
```

`Image`为内核文件,`Image.gz`为gzip压缩的内核,修改uboot或者其他引导器的配置文件可以启动这个内核

注意,这个内核没有initramfs,因此必须有启动参数ROOT才能找到根分区,比如`ROOT=/dev/nvme0n1p2`.ROOT的值的格式与/etc/fstab的格式相同,可以使用`ROOT=LABEL=xxx`或`ROOT=UUID=xxxxxxxxxxxxx`.如果这个分区可以被内核识别,同时存在`/sbin/init`,则`init`被启动.如果没有指定rootfs,或者无法识别`ROOT`的目标,则会输出:

```output
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

大多数情况下,内核可以直接找到`ROOT`的分区并启动,但是处于故障维护的考虑,我们可能需要一个initramfs.

关于如何制作initramfs的资料可以参考Gentoo的wiki:[Custom Initramfs](https://wiki.gentoo.org/wiki/Custom_Initramfs)

## 编译busybox

busybox将数十个符合POSIX的Unix工具集成到一个不到2M的可执行文件中,非常适合在内存中运行的initramfs环境.

```shell
cd $DISTDIR && wget -qO- https://busybox.net/downloads/busybox-1.31.1.tar.bz2 | tar xvj && cd busybox-1.31.1
```

将[此邮件列表](http://lists.busybox.net/pipermail/busybox/2019-June/087337.html)中的补丁保存为`clang.patch`

```shell
git apply clang.patch
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE menuconfig
# 选中Settings -> Don't use /usr 和 Build static binary (no shared libs)
make ARCH=$ARCH CFLAGS="-Wignored-optimization-argument ${CFLAGS}" CROSS_COMPILE=$CROSS_COMPILE -j7
mkdir $INITRAMFS/bin && cp busybox $INITRAMFS/bin
```

验证可以使用.

```shell
$file busybox
busybox: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, stripped
$qemu-aarch64 busybox uname -m
aarch64
```

## 构造initramfs

以root身份运行以下命令

```shell
# busybox的mkdir不支持mkdir $INITRAMFS/{bin,dev,proc,sbin,sys}
for i in bin dev proc sbin sys;do
  mkdir $INITRAMFS/$i
done
# cp --archive /dev/{null,console,tty} $INITRAMFS/dev/
for i in null console tty;do
  cp --archive /dev/$i $INITRAMFS/dev/
done
```

创建`$INITRAMFS/init`文件:

```shell
#!/bin/busybox sh
/bin/busybox --install -s
mount -t devtmpfs devtmpfs /dev
mount -t proc none /proc
mount -t sysfs none /sys
setsid cttyhack sh
poweroff -f
```

将其设置为可运行文件:`chmod +x init`

最终的文件列表为:

```txt
drwxr-xr-x    - root  ├── bin
.rwxr-xr-x 1.2M root  │  └── busybox
drwxr-xr-x    - root  ├── dev
crw-------  5,1 root  │  ├── console
crw-rw-rw-  1,3 root  │  ├── null
crw-rw-rw-  5,0 root  │  └── tty
.rwxr-xr-x  157 root  ├── init
drwxr-xr-x    - root  ├── proc
drwxr-xr-x    - root  ├── sbin
drwxr-xr-x    - root  └── sys
```

## 将initramfs打包进内核

进入内核设置界面:

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE menuconfig
```

```txt
    General setup  --->
 │       [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support                  │
 │           (../../initramfs) Initramfs source file(s)                                         │
 │           (0)   User ID to map to 0 (user root)                                           │
 │           (0)   Group ID to map to 0 (group root)                                         │
 │       [ ]   Support initial ramdisk/ramfs compressed using gzip                           │
 │       [ ]   Support initial ramdisk/ramfs compressed using bzip2                          │
 │       [ ]   Support initial ramdisk/ramfs compressed using LZMA                           │
 │       [ ]   Support initial ramdisk/ramfs compressed using XZ                             │
 │       [ ]   Support initial ramdisk/ramfs compressed using LZO                            │
 │       [*]   Support initial ramdisk/ramfs compressed using LZ4                            │
 │       [*]   Built-in initramfs compression mode (LZ4)  --->                               │
```

重新编译

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -j7
```

## 测试

可以在qemu或者真机上测试内核.

下面是一个最小的Linux 5.5.2 aarch64内核的启动过程,你可以参考这份[配置文件](https://gist.github.com/12101111/b6978a114c9acca23a67fd064e57f040)：

```log
$ qemu-system-aarch64 -machine virt -nographic -cpu cortex-a72 -kernel Image -m 32M
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 5.5.2 (root@omen) (clang version 10.0.0) #4 SMP Mon Feb 10 16:14:45 CST 2020
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] percpu: Embedded 20 pages/cpu s52320 r0 d29600 u81920
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: kernel page table isolation disabled by kernel configuration
[    0.000000] spectrev2 mitigation disabled by kernel configuration
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 8064
[    0.000000] Kernel command line:
[    0.000000] Dentry cache hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.000000] Inode-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 11584K/32768K available (1470K kernel code, 112K rwdata, 204K rodata, 1472K init, 241K bss, 21184K reserved, 0K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=1.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000161] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.001837] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.001989] pid_max: default: 4096 minimum: 301
[    0.006504] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.006555] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.034626] ASID allocator initialised with 65536 entries
[    0.035499] rcu: Hierarchical SRCU implementation.
[    0.038368] smp: Bringing up secondary CPUs ...
[    0.038475] smp: Brought up 1 node, 1 CPU
[    0.038579] SMP: Total of 1 processors activated.
[    0.038654] CPU features: detected: 32-bit EL0 Support
[    0.038736] CPU features: detected: CRC32 instructions
[    0.039961] CPU: All CPU(s) started at EL1
[    0.040146] alternatives: patching kernel code
[    0.051664] devtmpfs: initialized
[    0.060394] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.063040] DMA: preallocated 256 KiB pool for atomic allocations
[    0.065157] Serial: AMBA PL011 UART driver
[    0.087572] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 39, base_baud = 0) is a PL011 rev1
[    0.098202] printk: console [ttyAMA0] enabled
[    0.118475] clocksource: Switched to clocksource arch_sys_counter
[    0.188649] workingset: timestamp_bits=62 max_order=12 bucket_order=0
[    0.203966] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.205089] random: get_random_bytes called from 0xffff80001009249c with crng_init=0
[    0.228200] Freeing unused kernel memory: 1472K
[    0.231046] Run /init as init process
/ # ls
bin      etc      linuxrc  root     sys
dev      init     proc     sbin
/ # ls /dev/
console          kmsg             ptmx             ttyAMA0
cpu_dma_latency  mem              random           urandom
full             null             tty              zero
/ # uname -a
Linux (none) 5.5.2 #4 SMP Mon Feb 10 16:14:45 CST 2020 aarch64 GNU/Linux
/ # [   10.191195] reboot: Power down
```

按Ctrl-D或者输入exit来关机.
