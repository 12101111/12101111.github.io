+++
title = "LLVM cross-compiled Linux From Scratch: ZFS on root (Optional)"
date = 2020-08-26
[taxonomies]
categories = ["Linux"]
tags = ["Linux","LLVM","ZFS"]
+++

本文章是LLVM编译Linux系统的第三篇文章,以一个实际场景介绍如何使用LLVM/Clang/musl工具链编译出适合于启动根分区在ZFS文件系统的Linux系统.

ZFS原是Solaris操作系统的文件系统,由Open Solaris项目以CDDL协议开源,随后Oracle收购Sun,将Solaris转为闭源系统,原开发者以illumos为名继续开发开源的Solaris系统,并创建OpenZFS项目继续ZFS的开发.ZFS先后被移植到FreeBSD,Linux,macOS和Windows.

然而在Linux平台,由于CDDL开源协议和GPLv2协议都要求衍生代码不得更改许可证,因而互不兼容,
这导致分发包含ZFS on Linux的Linux内核存在法律风险.
同样的,由于有可能被Oracle起诉,Torvalds linus拒绝合并ZFS到Linux,除非Oracle以GPLv2重新发布ZFS的代码.
ZFS一直以树外代码的形式开发,正如nvidia这样的闭源驱动一样在用户计算机上通过DKMS编译成为内核模块.
但实际上用户可以将Linux和ZFS的代码混合,并将ZFS编译进内核而不是模块(注意,这样的内核不能分发给他人).

但即使将ZFS编译为内置模块,Linux也不能不使用initramfs直接从ZFS分区上启动(比如通过启动参数指定启动分区),[此问题有待日后解决](https://github.com/zfsonlinux/zfs/issues/4300),目前需要一个initramfs在用户空间挂载根分区.

本文将使用之前编译的LLVM工具链编译内核与initramfs工具.

<!-- more -->

## 面向读者

本文主要面向ZFS用户或潜在用户,希望从ZFS分区启动,希望自己编译Linux并制作initramfs.

本文应该只适合于x86_64架构,aarch64平台由于内联汇编的编译错误无法进行.

## ZFS on linux内核模块

```shell
cd $DISTDIR && wget -qO- https://github.com/openzfs/zfs/releases/download/zfs-2.0.0-rc1/zfs-2.0.0-rc1.tar.gz | tar xvz && cd zfs-2.0.0-rc1
```

如果在Alpine中, 打1个补丁：

```shell
sed -i 's/KERNEL_DIR="$(readlink --canonicalize-existing "$1")"/KERNEL_DIR="$1"/g' copy-builtin
```

另一个补丁:

```diff
diff --git a/config/kernel.m4 b/config/kernel.m4
index ec52f014a..4b2f31cc3 100644
--- a/config/kernel.m4
+++ b/config/kernel.m4
@@ -571,7 +571,7 @@ dnl #
 AC_DEFUN([ZFS_LINUX_COMPILE], [
        AC_TRY_COMMAND([
            KBUILD_MODPOST_NOFINAL="$5" KBUILD_MODPOST_WARN="$6"
-           make modules -k -j$TEST_JOBS -C $LINUX_OBJ $ARCH_UM
+           make CC=$CC modules -k -j$TEST_JOBS -C $LINUX_OBJ $ARCH_UM
            M=$PWD/$1 >$1/build.log 2>&1])
        AS_IF([AC_TRY_COMMAND([$2])], [$3], [$4])
 ])
```

配置zfs以生成配置文件,使用zfs提供的脚本将内核源码复制到Linux根目录,然后重新编译Linux.

```shell
./configure --host $TARGET --enable-linux-builtin --with-config=kernel --with-linux=$DISTDIR/linux
./copy-builtin $DISTDIR/linux
cd $DISTDIR/linux
echo 'CONFIG_ZFS=y' >> .config
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -j4
```

## ZFS 管理工具

zfs需要4个依赖库,将这些库编译为动态库.

### libtirpc

tirpc是SUN rpc(远程过程调用)的开源实现.tirpc需要一个纯C语言头文件实现的队列操作库,集成于glibc,
但是由于不属于任何C 标准,没有在musl中集成.NetBSD有一个便携的实现,需要下载到sysroot.

```shell
cd $DISTDIR && wget -qO- https://jaist.dl.sourceforge.net/project/libtirpc/libtirpc/1.2.6/libtirpc-1.2.6.tar.bz2 | tar xvj && cd libtirpc-1.2.6
./configure --host=x86_64-linux-musl --prefix= --disable-gssapi --disable-ipv6 --disable-static
wget https://raw.githubusercontent.com/NetBSD/src/trunk/sys/sys/queue.h -qO $SYSROOT/include/sys/queue.h
make -j7
DESTDIR=$SYSROOT make install
```

### zlib

zlib是一个广泛使用的压缩库.

```shell
cd $DISTDIR && wget -qO- https://nchc.dl.sourceforge.net/project/libpng/zlib/1.2.11/zlib-1.2.11.tar.xz | tar xvJ && cd zlib-1.2.11
CC=x86_64-linux-musl-clang ./configure --prefix= --shared
make
DESTDIR=$SYSROOT make install
```

### libressl

libressl是OpenBSD维护的OpenSSL的fork.我们需要其中的libcrypto和libssl用于加密解密操作.

```shell
cd $DISTDIR && wget -qO- https://ftp.openbsd.org/pub/OpenBSD/LibreSSL/libressl-3.2.1.tar.gz | tar xvz && cd libressl-3.2.1
mkdir build && cd build && cmake ../ -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_C_COMPILER_TARGET=$TARGET -DCMAKE_SYSROOT=$SYSROOT -DCMAKE_INSTALL_PREFIX=$SYSROOT -DCMAKE_INSTALL_LIBDIR="lib" -DLIBRESSL_APPS=OFF -DLIBRESSL_TESTS=OFF -DBUILD_SHARED_LIBS=ON
ninja
ninja install
```

### util-linux

util-linux是包含很多linux专有的(而非是POSIX或其他Unxi-like也有的)工具.我们不需要这些工具(很多包含于busybox中),而是其中的两个库:`libblkid`,用于获取磁盘的信息,`libuuid`,用于生成UUID.

```shell
cd $DISTDIR && wget -qO- https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.36/util-linux-2.36.tar.xz | tar xvJ && cd util-linux-2.36
./configure --host=x86_64-linux-musl --prefix= --disable-static --disable-all-programs --enable-libblkid --enable-libuuid
make -j7
DESTDIR=$SYSROOT make install
```

### zfs on linux 工具

最终编译zfs的工具,我们主要需要其中的`zpool`和`zfs`命令.

```shell
cd $DISTDIR/zfs-2.0.0-rc1
export PKG_CONFIG_LIBDIR=$SYSROOT/lib/pkgconfig
export PKG_CONFIG_SYSROOT_DIR=$SYSROOT
./configure --prefix= --with-config=user --host=x86_64-linux-musl --disable-systemd --disable-sysvinit  --without-udevdir --without-udevruledir --disable-nls --disable-static --disable-pyzfs --disable-rpath
make -j7
DESTDIR=$SYSROOT make install
```

## 修改initramfs

将编译好的zfs可执行程序复制到对应目录,并移除其`rpath`。

```shell
for i in zfs zdb zpool mount.zfs;do
  cp $SYSROOT/sbin/$i $INITRAMFS/sbin/
  patchelf --remove-rpath $INITRAMFS/sbin/$i
done
```

查看这些程序的依赖

```shell
$ readelf -d $INITRAMFS/sbin/* | grep "Shared library" | sort | uniq
  0x0000000000000001 (NEEDED)             Shared library: [libblkid.so.1]
  0x0000000000000001 (NEEDED)             Shared library: [libc.so]
  0x0000000000000001 (NEEDED)             Shared library: [libcrypto.so.46]
  0x0000000000000001 (NEEDED)             Shared library: [libnvpair.so.1]
  0x0000000000000001 (NEEDED)             Shared library: [libtirpc.so.3]
  0x0000000000000001 (NEEDED)             Shared library: [libuuid.so.1]
  0x0000000000000001 (NEEDED)             Shared library: [libuutil.so.1]
  0x0000000000000001 (NEEDED)             Shared library: [libz.so.1]
  0x0000000000000001 (NEEDED)             Shared library: [libzfs.so.2]
  0x0000000000000001 (NEEDED)             Shared library: [libzfs_core.so.1]
  0x0000000000000001 (NEEDED)             Shared library: [libzpool.so.2]
```

使用命令复制这些依赖：

```shell
for i in $(readelf -d $INITRAMFS/sbin/* | grep "Shared library" | sort | uniq | sed 's/^[ ]*0x[01]* (NEEDED)[ ]*Shared library: \[\(lib[a-z_]*.so\.\?[0-9]*\)\].*/\1/g'); do
    cp $SYSROOT/lib/$i $INITRAMFS/lib
done
```

## 配置ZFS文件系统

在大多数情况下,系统的`/{etc,bin,sbin,lib}`必须在initramfs中挂载,这样真正的init(OpenRC,SysV RC或systemd)才能继续执行启动过程.init也应该启用ZFS提供的挂载服务来挂载所有的挂载点.
如果`/sbin/zfs`依赖`/usr/lib`中的库,则`/usr/lib`也应该在initramfs中挂载,
否则会出现`zfs`无法执行的问题.下面是我的配置,仅供参考.
在initramfs中挂载的dataset的`mountpoint`属性应设置为`legacy`,
否则只能使用`zfs mount`命令挂载到真正的挂载点而不是相对于`newroot`的挂载点.

```output
$ zfs list
NAME             USED  AVAIL     REFER  MOUNTPOINT
data             228G  97.9G      104K  /data
data/binpkgs    2.46G  97.9G     2.46G  /var/cache/binpkgs
data/ccache     23.8G  97.9G     23.8G  /var/cache/ccache
data/distfiles  9.36G  97.9G     9.36G  /var/cache/distfiles
data/gentoo     7.10G  97.9G     7.10G  legacy
data/home        167G  97.9G      167G  /home
data/musl       13.2G  97.9G     12.4G  legacy
data/portage    1.42G  97.9G     1.32G  /usr/portage
data/src        3.63G  97.9G     3.63G  /usr/src
data/tmp         104K  97.9G      104K  /usr/tmp
```

修改init.请根据实际情况调整挂载dataset的命令.

```shell
#!/bin/busybox sh

export BOX="/bin/busybox"
export PATH="/bin:/sbin"

rescue_shell() {
    echo "Something went wrong: fail to $1. Dropping to a shell."
    [ -e /proc/cmdline ] || $BOX mount -t proc none /proc || echo "Fatal error: unable to mount procfs"
    [ -d /sys/class ] || $BOX mount -t sysfs none /sys || echo "Fatal error: unable to mount sysfs"
    $BOX mkdir -p /usr/bin
    $BOX mkdir -p /usr/sbin
    export PATH="/bin:/sbin:/usr/bin:/usr/sbin"
    $BOX --install -s
    setsid cttyhack ash
    echo "Now poweroff."
    sleep 10
    poweroff -f
}

cmdline() {
    local value
    value=" $(cat /proc/cmdline) "
    value="${value##* ${1}=}"
    value="${value%% *}"
    [ -n "${value}" ] && echo "${value}"
}

echo "==> Setup initramfs"

$BOX mount -t proc none /proc || rescue_shell "mount procfs"
$BOX mount -t sysfs none /sys || rescue_shell "mount sysfs"
$BOX mount -t devtmpfs devtmpfs /dev || rescue_shell "mount devtmpfs"
$BOX mkdir /dev/pts || rescue_shell "mkdir /dev/pts"
$BOX mount -t devpts /dev/pts /dev/pts || rescue_shell "mount devpts"

echo 0 > /proc/sys/kernel/printk

if [ -n "$(cmdline failsafe)" ]; then
    rescue_shell "boot"
fi

echo "==> Import ZFS pool"
export ZPOOL_IMPORT_UDEV_TIMEOUT_MS=0
zpool import -Naf || rescue_shell "import zpool"

echo "zpool list:"
zpool list || rescue_shell "list zpool"

echo "==> Mount root dataset"

export ROOT="$(cmdline ROOT)"

echo "kernel cmdline: ROOT=${ROOT}"

[ -z "${ROOT}" ] && export ROOT="gentoo"

echo "use dataset: /data/$ROOT"

$BOX mkdir /newroot || rescue_shell "mkdir /newroot"
mount.zfs data/$ROOT /newroot || rescue_shell "mount /"
mount.zfs data/$ROOT/usr /newroot/usr || rescue_shell "mount /usr"
mount.zfs data/$ROOT/var /newroot/var || rescue_shell "mount /var"

echo "==> Clean up"
$BOX umount /sys || rescue_shell "umount /sys"
$BOX umount /proc || rescue_shell "umount /proc"

echo "==> Switch root and execute init"
exec $BOX switch_root /newroot /sbin/init || rescue_shell "switch_root"

rescue_shell "Failed to run /sbin/init"
```

最终的文件列表为:

```txt
.
├── bin
│  └── busybox
├── dev
│  ├── console
│  ├── null
│  ├── random
│  ├── tty
│  ├── urandom
│  └── zero
├── etc
│  └── mtab -> /proc/self/mounts
├── init
├── lib
│  ├── ld-musl-x86_64.so.1 -> libc.so
│  ├── libblkid.so.1
│  ├── libc.so
│  ├── libcrypto.so.46
│  ├── libnvpair.so.1
│  ├── libssl.so.1.1
│  ├── libtirpc.so.3
│  ├── libudev.so.1
│  ├── libuuid.so.1
│  ├── libuutil.so.1
│  ├── libz.so.1
│  ├── libzfs.so.2
│  ├── libzfs_core.so.1
│  └── libzpool.so.2
├── proc
├── sbin
│  ├── mount.zfs
│  ├── zdb
│  ├── zfs
│  └── zpool
└── sys
```

重新编译Linux内核.

```shell
cd $DISTDIR/linux
rm usr/initramfs_data.cpio* built-in.a
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -j13
```

这样,编译完的Linux内核就可以直接从UEFI Shell中启动,并启动ZFS中的系统.
如果启动失败,就会进入busybox的shell界面,可以使用`zfs`命令手动挂载根文件系统到`/newroot`,
然后手动执行`exec switch_root /newroot /sbin/init`
