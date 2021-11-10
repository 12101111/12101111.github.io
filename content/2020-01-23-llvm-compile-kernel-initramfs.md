+++
title = "LLVM cross-compiled Linux From Scratch: Bootable kernel and initramfs"
date = 2021-11-10
[taxonomies]
categories = ["Linux"]
tags = ["Linux","LLVM"]
+++

本文章是LLVM编译Linux系统的第二篇文章,介绍如何使用LLVM/Clang/musl工具链编译一个可以启动的Linux内核.

<!-- more -->

## 编译GNU assembler

大多数情况下LLVM提供的binutils已经兼容GNU binutils了,但是少数情况下, LLVM/Clang的内置汇编器的行为与GNU汇编器并不相同, 如果遇到编译Linux内核时Clang内置汇编器报错的情况,则需要使用GNU汇编器. ClangBuiltLinux项目为Clang编译Linux内核做出了大量的努力, 如果遇到此类错误, 可以去[ClangBuiltLinux项目的issue区](https://github.com/ClangBuiltLinux/linux/issues?q=is%3Aopen+is%3Aissue+label%3A%22%5BTOOL%5D+integrated-as%22)查看问题是否已知.

```shell
cd $DISTDIR && wget -qO- https://mirrors.tuna.tsinghua.edu.cn/gnu/binutils/binutils-2.37.tar.xz | tar xvJ && cd binutils-2.37
unset CFLAGS CXXFLAGS LDFLAGS #这部分是为Host编译的
export acx_cv_cc_gcc_supports_ada=no # 如果你的gcc/cc是clang的链接则需要此环境变量
./configure --prefix= --target=$TARGET --disable-nls --disable-werror --disable-ld --disable-gold --disable-gprof --disable-binutils
make -j13
DESTDIR=$HOME/.local/ make install
```

## 编译Linux内核

打开设置菜单

```shell
export ARCH=arm64 # 对应内核arch目录的子目录名/
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 menuconfig
```

请自行根据硬件调整配置,可以参考[Gentoo的文档](https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide)

总的来说,可以通过启动livecd来判断所需要的内核组件.对于计算机启动时就会加载的模块,选择`[Y]`,对于USB设备则可以选择为`[M]`,其他用不到的选择为`[N]`.

对于x86_64 UEFI启动的计算机,建议编译时打开`CONFIG_EFI_STUB`选项,这样就可以直接从UEFI shell启动了.

对于树梅派3/4等arm设备,树莓派的fork的内核源码提供预先编写的配置.

```shell
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 bcmrpi3_defconfig # Pi 2 (1.2+), Pi 3, Pi 3+, or Compute Module 3
make ARCH=arm64 CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 bcm2711_defconfig # Pi 4b+
```

如果只是想在虚拟机中测试的话，可以选择：

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 tinyconfig
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 kvm_guest.config
# 启用 General setup --> Configure standard kernel features (expert users) --> Multiple users, groups and capabilities support
# 和 Enable support for printk
# 启用 Device Drivers --> Character devices --> Serial drivers --> ARM AMBA PL011 serial port support 
# 和 Support for console on AMBA serial port ( 如果你使用arm架构 )
# 启用 Kernel hacking --> printk and dmesg options --> Show timing information on printks
# 启用 File systems --> Pseudo filesystems --> /proc file system support
# 启用 File systems --> Pseudo filesystems --> sysfs file system support
# 启用 Executable file firmats --> Kernel support for scripts starting with #!
# 启用 Device Drivers --> Generic Driver Options --> Maintain a devtmpfs filesystem to mount at /dev
```

然后编译内核.

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 -j13
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

对于arm64系统会有以下输出

```output
  LD      vmlinux
  SORTEX  vmlinux
  SYSMAP  System.map
  OBJCOPY arch/arm64/boot/Image
  GZIP    arch/arm64/boot/Image.gz
```

`Image`为内核文件,`Image.gz`为gzip压缩的内核,修改uboot或者其他引导器的配置文件可以启动这个内核, 也可以直接用qemu启动:

```output
> qemu-system-aarch64 -m 64M -M virt -cpu cortex-a53 -kernel arch/arm64/boot/Image -nographic -append "console=ttyAMA0"
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 5.15.1-zen+ (han@musl) (clang version 13.0.0, LLD 13.0.0) #110 ZEN SMP Tue Nov 9 17:40:11 CST 2021
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000040000000-0x0000000043ffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x0000000043ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x0000000043ffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] percpu: Embedded 16 pages/cpu s33688 r0 d31848 u65536
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: kernel page table isolation disabled by kernel configuration
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 16160
[    0.000000] Kernel command line: console=ttyAMA0
[    0.000000] Dentry cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000000] Inode-cache hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] Memory: 57516K/65536K available (3072K kernel code, 610K rwdata, 340K rodata, 320K init, 255K bss, 8020K reserved, 0K cma-reserved)
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=1.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] Root IRQ handler: 0xffffffc010145db4
[    0.000000] GICv2m: range[mem 0x08020000-0x08020fff], SPI[80:143]
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000050] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.001878] Console: colour dummy device 80x25
[    0.002848] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.002995] pid_max: default: 4096 minimum: 301
[    0.005089] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.005122] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.019348] rcu: Hierarchical SRCU implementation.
[    0.021194] smp: Bringing up secondary CPUs ...
[    0.021303] smp: Brought up 1 node, 1 CPU
[    0.021331] SMP: Total of 1 processors activated.
[    0.021372] CPU features: detected: 32-bit EL0 Support
[    0.021431] CPU features: detected: CRC32 instructions
[    0.022349] CPU: All CPU(s) started at EL1
[    0.022470] alternatives: patching kernel code
[    0.027233] random: get_random_bytes called from 0xffffffc01038a914 with crng_init=0
[    0.028666] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.030098] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.032806] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.035296] ASID allocator initialised with 65536 entries
[    0.035330] Serial: AMBA PL011 UART driver
[    0.049279] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 47, base_baud = 0) is a PL011 rev1
[    0.057230] printk: console [ttyAMA0] enabled
[    0.061698] vgaarb: loaded
[    0.069121] clocksource: Switched to clocksource arch_sys_counter
[    0.072978] NET: Registered PF_INET protocol family
[    0.074091] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.075695] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.075954] TCP established hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.076237] TCP bind hash table entries: 512 (order: 1, 8192 bytes, linear)
[    0.076530] TCP: Hash tables configured (established 512 bind 512)
[    0.077986] UDP hash table entries: 128 (order: 0, 4096 bytes, linear)
[    0.078397] UDP-Lite hash table entries: 128 (order: 0, 4096 bytes, linear)
[    0.079387] PCI: CLS 0 bytes, default 64
[    0.083992] workingset: timestamp_bits=62 max_order=14 bucket_order=0
[    0.084285] 9p: Installing v9fs 9p2000 file system support
[    0.084853] io scheduler mq-deadline registered
[    0.085207] io scheduler kyber registered
[    0.098168] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.100453] NET: Registered PF_INET6 protocol family
[    0.105579] Segment Routing with IPv6
[    0.105776] In-situ OAM (IOAM) with IPv6
[    0.105978] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    0.108356] 9pnet: Installing 9P2000 support
[    0.117062] List of all partitions:
[    0.117262] No filesystem could mount root, tried:
[    0.117277]
[    0.117571] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    0.117983] Kernel Offset: disabled
[    0.118090] CPU features: 0x00000000,00000802
[    0.118393] Memory Limit: none
[    0.118767] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

注意,这个内核没有initramfs,因此必须有启动参数ROOT才能找到根分区,比如`ROOT=/dev/nvme0n1p2`.ROOT的值的格式与/etc/fstab的格式相同,可以使用`ROOT=LABEL=xxx`或`ROOT=UUID=xxxxxxxxxxxxx`.如果这个分区可以被内核识别,同时存在`/sbin/init`,则`init`被启动.如果没有指定rootfs,或者无法识别`ROOT`的目标,则会输出:

```output
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

大多数情况下,内核可以直接找到`ROOT`的分区并启动,但是处于故障维护的考虑,我们可能需要一个initramfs.

关于如何制作initramfs的资料可以参考Gentoo的wiki:[Custom Initramfs](https://wiki.gentoo.org/wiki/Custom_Initramfs)

## 编译busybox

busybox将数十个符合POSIX的Unix工具集成到一个不到2M的可执行文件中,非常适合在内存中运行的initramfs环境.

```shell
cd $DISTDIR && wget -qO- https://busybox.net/downloads/busybox-1.34.1.tar.bz2 | tar xvj && cd busybox-1.34.1

# 打一个补丁, 否则clang编译出来的程序会直接崩溃
wget -qO- https://github.com/kraj/meta-clang/raw/master/recipes-core/busybox/busybox/0001-Turn-ptr_to_globals-and-bb_errno-to-be-non-const.patch | patch -p1

make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE menuconfig
# 选中Settings -> Don't use /usr 和 Build static binary (no shared libs)
make ARCH=$ARCH CFLAGS="${CFLAGS}" CROSS_COMPILE=$CROSS_COMPILE -j13
mkdir $INITRAMFS/bin && cp busybox $INITRAMFS/bin
```

验证可以使用.

```shell
> file busybox
busybox: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, stripped
> qemu-aarch64 busybox uname -m
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

initramfs可以分开打包, 也可以包含在内核内.

如果想分开打包,进入内核设置界面

```
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE LLVM=1 LLVM_IAS=1 menuconfig
```

来到General Setup, 选中`Initial RAM filesystem and RAM disk (initramfs/initrd) support`, 以及下面想要使用的压缩格式.

ZSTD压缩率最高且解压速度快, 建议实际部署时使用. gzip最简单, 测试时可以用.

```shell
cd $INITRAMFS && find . 2>/dev/null | cpio -ov -R root:root -H newc | gzip -9 > ../initramfs.img.gz
```

```
[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
()    Initramfs source file(s) (NEW)
[*]   Support initial ramdisk/ramfs compressed using gzip (NEW)
[ ]   Support initial ramdisk/ramfs compressed using bzip2
[ ]   Support initial ramdisk/ramfs compressed using LZMA
[ ]   Support initial ramdisk/ramfs compressed using XZ
[ ]   Support initial ramdisk/ramfs compressed using LZO
[ ]   Support initial ramdisk/ramfs compressed using LZ4
[ ]   Support initial ramdisk/ramfs compressed using ZSTD 
```

使用如下命令打包出initramfs镜像压缩包.

```shell
cd $INITRAMFS && find . 2>/dev/null | cpio -ov -R root:root -H newc | gzip -9 > ../initramfs.img.gz
```

如果想打包进内核, 进入内核设置界面, 在Initramfs source file(s)选择`$INITRAMFS`的地址

```txt
[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
(/home/han/aarch64-linux-musl/initramfs) Initramfs source file(s)
(0)   User ID to map to 0 (user root) (NEW)
(0)   Group ID to map to 0 (group root) (NEW)
[*]   Support initial ramdisk/ramfs compressed using gzip
[ ]   Support initial ramdisk/ramfs compressed using bzip2
[ ]   Support initial ramdisk/ramfs compressed using LZMA
[ ]   Support initial ramdisk/ramfs compressed using XZ
[ ]   Support initial ramdisk/ramfs compressed using LZO
[ ]   Support initial ramdisk/ramfs compressed using LZ4
[ ]   Support initial ramdisk/ramfs compressed using ZSTD
Built-in initramfs compression mode (Gzip)  ---> 
```

重新编译内核

```shell
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -j13
```

## 测试

可以在qemu或者真机上测试内核.

```log
> qemu-system-aarch64 -m 64M -M virt -cpu cortex-a53 -kernel ./Image.gz -initrd ./initramfs.img.gz -nographic -append "console=ttyAMA0"
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 5.15.1-zen+ (han@musl) (clang version 13.0.0, LLD 13.0.0) #115 ZEN SMP Tue Nov 9 19:27:33 CST 2021
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000040000000-0x0000000043ffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x0000000043ffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x0000000043ffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] percpu: Embedded 16 pages/cpu s34392 r0 d31144 u65536
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: kernel page table isolation disabled by kernel configuration
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 16160
[    0.000000] Kernel command line: console=ttyAMA0
[    0.000000] Dentry cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000000] Inode-cache hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] Memory: 55076K/65536K available (3328K kernel code, 654K rwdata, 412K rodata, 1344K init, 260K bss, 10460K reserved, 0K cma-reserved)
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=1.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] Root IRQ handler: 0xffffffc010169f08
[    0.000000] GICv2m: range[mem 0x08020000-0x08020fff], SPI[80:143]
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (virt).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000052] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.001858] Console: colour dummy device 80x25
[    0.002812] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.002927] pid_max: default: 4096 minimum: 301
[    0.004977] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.005011] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.024230] rcu: Hierarchical SRCU implementation.
[    0.026300] smp: Bringing up secondary CPUs ...
[    0.026386] smp: Brought up 1 node, 1 CPU
[    0.026412] SMP: Total of 1 processors activated.
[    0.026454] CPU features: detected: 32-bit EL0 Support
[    0.026513] CPU features: detected: CRC32 instructions
[    0.027487] CPU: All CPU(s) started at EL1
[    0.027609] alternatives: patching kernel code
[    0.035079] devtmpfs: initialized
[    0.040810] random: get_random_bytes called from 0xffffffc0103dd960 with crng_init=0
[    0.042477] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.043860] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.047242] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.049753] ASID allocator initialised with 65536 entries
[    0.049816] Serial: AMBA PL011 UART driver
[    0.067495] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 47, base_baud = 0) is a PL011 rev1
[    0.075982] printk: console [ttyAMA0] enabled
[    0.092989] vgaarb: loaded
[    0.101630] clocksource: Switched to clocksource arch_sys_counter
[    0.113153] NET: Registered PF_INET protocol family
[    0.114575] IP idents hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.117043] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.117523] TCP established hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.117866] TCP bind hash table entries: 512 (order: 1, 8192 bytes, linear)
[    0.118189] TCP: Hash tables configured (established 512 bind 512)
[    0.119488] UDP hash table entries: 128 (order: 0, 4096 bytes, linear)
[    0.119748] UDP-Lite hash table entries: 128 (order: 0, 4096 bytes, linear)
[    0.120968] PCI: CLS 0 bytes, default 64
[    0.138685] workingset: timestamp_bits=62 max_order=14 bucket_order=0
[    0.139068] 9p: Installing v9fs 9p2000 file system support
[    0.139586] io scheduler mq-deadline registered
[    0.139796] io scheduler kyber registered
[    0.186347] Unpacking initramfs...
[    0.252316] Freeing initrd memory: 1008K
[    0.274845] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.277940] NET: Registered PF_INET6 protocol family
[    0.284542] Segment Routing with IPv6
[    0.284787] In-situ OAM (IOAM) with IPv6
[    0.285422] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    0.289354] 9pnet: Installing 9P2000 support
[    0.312361] Freeing unused kernel memory: 1344K
[    0.321870] Run /init as init process
/ # uname -a
Linux (none) 5.15.1-zen+ #115 ZEN SMP Tue Nov 9 19:27:33 CST 2021 aarch64 GNU/Linux
/ # ps aux
PID   USER     TIME  COMMAND
    1 0         0:00 {init} /bin/busybox sh /init
    2 0         0:00 [kthreadd]
    3 0         0:00 [rcu_gp]
    4 0         0:00 [rcu_par_gp]
    5 0         0:00 [kworker/0:0-eve]
    6 0         0:00 [kworker/0:0H-ev]
    7 0         0:00 [kworker/u2:0-ev]
    8 0         0:00 [mm_percpu_wq]
    9 0         0:00 [ksoftirqd/0]
   10 0         0:00 [rcu_sched]
   11 0         0:00 [migration/0]
   12 0         0:00 [cpuhp/0]
   13 0         0:00 [kdevtmpfs]
   14 0         0:00 [inet_frag_wq]
   15 0         0:00 [oom_reaper]
   16 0         0:00 [writeback]
   17 0         0:00 [kblockd]
   18 0         0:00 [kworker/0:1-eve]
   19 0         0:00 [kworker/0:1H]
   20 0         0:00 [kworker/u2:1]
   21 0         0:00 [kswapd0]
   22 0         0:00 [mld]
   23 0         0:00 [ipv6_addrconf]
   28 0         0:00 sh
   30 0         0:00 ps aux
/ # free -h
              total        used        free      shared  buff/cache   available
Mem:          56.1M        5.6M       47.1M           0        3.4M       45.8M
Swap:             0           0           0
/ # cat /proc/version
Linux version 5.15.1-zen+ (han@musl) (clang version 13.0.0, LLD 13.0.0) #115 ZEN SMP Tue Nov 9 19:27:33 CST 2021
/ # cat /proc/cpuinfo
processor	: 0
BogoMIPS	: 125.00
Features	: fp asimd aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd03
CPU revision	: 4
/ # ls /dev/
console      tty14        tty27        tty4         tty52        tty8
full         tty15        tty28        tty40        tty53        tty9
kmsg         tty16        tty29        tty41        tty54        ttyAMA0
null         tty17        tty3         tty42        tty55        urandom
port         tty18        tty30        tty43        tty56        vcs
ptmx         tty19        tty31        tty44        tty57        vcs1
random       tty2         tty32        tty45        tty58        vcsa
tty          tty20        tty33        tty46        tty59        vcsa1
tty0         tty21        tty34        tty47        tty6         vcsu
tty1         tty22        tty35        tty48        tty60        vcsu1
tty10        tty23        tty36        tty49        tty61        vga_arbiter
tty11        tty24        tty37        tty5         tty62        zero
tty12        tty25        tty38        tty50        tty63
tty13        tty26        tty39        tty51        tty7
/ # ls /proc/
1             21            buddyinfo     iomem         self
10            22            bus           ioports       softirqs
11            23            cmdline       irq           stat
12            24            consoles      kmsg          swaps
13            29            cpuinfo       kpagecount    sys
14            3             device-tree   kpageflags    thread-self
15            31            devices       loadavg       timer_list
16            4             diskstats     meminfo       tty
17            5             driver        misc          uptime
18            6             execdomains   mounts        version
19            7             filesystems   net           vmallocinfo
2             8             fs            pagetypeinfo  vmstat
20            9             interrupts    partitions    zoneinfo
/ # ls /sys/
block     class     devices   fs        module
bus       dev       firmware  kernel
/ # exit
[   14.156869] reboot: Power down
```

按Ctrl-D或者输入exit来关机.
