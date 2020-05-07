+++
title = "LLVM cross-compiled Linux From Scratch: ZFS on root (Optional)"
date = 2020-02-08
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
cd $DISTDIR && wget -qO- https://github.com/zfsonlinux/zfs/releases/download/zfs-0.8.3/zfs-0.8.3.tar.gz | tar xvz && cd zfs-0.8.3
```

打几个补丁：

```shell
# https://github.com/zfsonlinux/zfs/pull/9927
sed -i 's/CFLAGS="$CFLAGS/CFLAGS="$CFLAGS -Werror/g' config/always-compiler-options.m4
./autogen.sh
# 如果在Alpine中
sed -i 's/KERNEL_DIR="$(readlink --canonicalize-existing "$1")"/KERNEL_DIR="$1"/g' copy-builtin
```

另一个补丁:

```diff
--- a/module/zfs/dbuf.c
+++ b/module/zfs/dbuf.c
@@ -2397,11 +2397,11 @@ dmu_buf_set_crypt_params(dmu_buf_t *db_fake, boolean_t byteorder,
        bcopy(mac, dr->dt.dl.dr_mac, ZIO_DATA_MAC_LEN);
 }

-#pragma weak dmu_buf_fill_done = dbuf_fill_done
 /* ARGSUSED */
 void
-dbuf_fill_done(dmu_buf_impl_t *db, dmu_tx_t *tx)
+dmu_buf_fill_done(dmu_buf_t *dbuf, dmu_tx_t *tx)
 {
+       dmu_buf_impl_t *db = (dmu_buf_impl_t *)dbuf;
        mutex_enter(&db->db_mtx);
        DBUF_VERIFY(db);
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
cd $DISTDIR && wget -qO- https://nchc.dl.sourceforge.net/project/libtirpc/libtirpc/1.2.5/libtirpc-1.2.5.tar.bz2 | tar xvj && cd libtirpc-1.2.5
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
cd $DISTDIR && wget -qO- https://ftp.openbsd.org/pub/OpenBSD/LibreSSL/libressl-3.0.2.tar.gz | tar xvz && cd libressl-3.0.2
mkdir build && cd build && cmake ../ -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_EXE_LINKER_FLAGS="$LDFLAGS" -DCMAKE_C_COMPILER_TARGET=$TARGET -DCMAKE_SYSROOT=$SYSROOT -DCMAKE_INSTALL_PREFIX=$SYSROOT -DLIBRESSL_APPS=OFF -DLIBRESSL_TESTS=OFF -DBUILD_SHARED_LIBS=ON
ninja
ninja install
```

### util-linux

util-linux是包含很多linux专有的(而非是POSIX或其他Unxi-like也有的)工具.我们不需要这些工具(很多包含于busybox中),而是其中的两个库:`libblkid`,用于获取磁盘的信息,`libuuid`,用于生成UUID.

```shell
cd $DISTDIR && wget -qO- https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.34/util-linux-2.34.tar.xz | tar xvJ && cd util-linux-2.34
./configure --host=x86_64-linux-musl --prefix= --disable-static --disable-all-programs --enable-libblkid --enable-libuuid
make -j7
DESTDIR=$SYSROOT make install
```

### zfs on linux 工具

最终编译zfs的工具,我们主要需要其中的`zpool`和`zfs`命令.

```shell
cd $DISTDIR/zfs-0.8.3
export PKG_CONFIG_PATH=$SYSROOT/lib/pkgconfig
export PKG_CONFIG_SYSROOT_DIR=$SYSROOT
./configure --prefix= --with-config=user --host=x86_64-linux-musl --disable-systemd --disable-sysvinit  --without-udevdir --without-udevruledir --disable-nls --disable-static --disable-pyzfs
make -j7
DESTDIR=$SYSROOT make install
```

## 修改initramfs

将编译好的库文件和zfs可执行程序复制到对应目录

```shell
mkdir $INITRAMFS/lib
cp -P $SYSROOT/lib/ld-musl-x86_64.so.1 $INITRAMFS/lib/
for i in blkid c crypto nvpair ssl tirpc uuid uutil z zfs zfs_core zpool;do
  cp -P $SYSROOT/lib/lib$i.so* $INITRAMFS/lib/
done
for i in zfs zdb zpool mount.zfs;do
  cp $SYSROOT/sbin/$i $INITRAMFS/sbin/
done
mkdir $INITRAMFS/newroot
```

## 配置ZFS文件系统

在大多数情况下,系统的`/{etc,bin,sbin,lib}`必须在initramfs中挂载,这样真正的init(OpenRC,SysV RC或systemd)才能继续执行启动过程.init也应该启用ZFS提供的挂载服务来挂载所有的挂载点.
如果`/sbin/zfs`依赖`/usr/lib`中的库,则`/usr/lib`也应该在initramfs中挂载,
否则会出现`zfs`无法执行的问题.下面是我的配置,仅供参考.
在initramfs中挂载的dataset的`mountpoint`属性应设置为`legacy`,
否则只能使用`zfs mount`命令挂载到真正的挂载点而不是相对于`newroot`的挂载点.

```output
$ zfs list
NAME                    USED  AVAIL     REFER  MOUNTPOINT
data                    241G   220G       25K  /data
data/gentoo            9.85G   220G      167M  legacy
data/gentoo/usr        4.45G   220G     2.79G  legacy
data/gentoo/usr/src    1.65G   220G     1.65G  /usr/src
data/gentoo/var        5.24G   220G     1.19G  /var
data/gentoo/var/cache  4.05G   220G     4.05G  /var/cache
data/home              37.1G   220G     37.1G  /home
data/linuxswap         8.50G   228G     14.8M  -
...
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
├── bin
│  └── busybox
├── dev
│  ├── console
│  ├── null
│  └── tty
├── init
├── lib
│  ├── ld-musl-x86_64.so.1 -> /lib/libc.so
│  ├── libblkid.so -> libblkid.so.1.1.0
│  ├── libblkid.so.1 -> libblkid.so.1.1.0
│  ├── libblkid.so.1.1.0
│  ├── libc.so
│  ├── libcrypto.so -> libcrypto.so.45
│  ├── libcrypto.so.45 -> libcrypto.so.45.5.0
│  ├── libcrypto.so.45.5.0
│  ├── libnvpair.so -> libnvpair.so.1.0.1
│  ├── libnvpair.so.1 -> libnvpair.so.1.0.1
│  ├── libnvpair.so.1.0.1
│  ├── libssl.so -> libssl.so.47
│  ├── libssl.so.47 -> libssl.so.47.6.0
│  ├── libssl.so.47.6.0
│  ├── libtirpc.so -> libtirpc.so.3.0.0
│  ├── libtirpc.so.3 -> libtirpc.so.3.0.0
│  ├── libtirpc.so.3.0.0
│  ├── libuuid.so -> libuuid.so.1.3.0
│  ├── libuuid.so.1 -> libuuid.so.1.3.0
│  ├── libuuid.so.1.3.0
│  ├── libuutil.so -> libuutil.so.1.0.1
│  ├── libuutil.so.1 -> libuutil.so.1.0.1
│  ├── libuutil.so.1.0.1
│  ├── libz.so -> libz.so.1.2.11
│  ├── libz.so.1 -> libz.so.1.2.11
│  ├── libz.so.1.2.11
│  ├── libzfs.so -> libzfs.so.2.0.0
│  ├── libzfs.so.2 -> libzfs.so.2.0.0
│  ├── libzfs.so.2.0.0
│  ├── libzfs_core.so -> libzfs_core.so.1.0.0
│  ├── libzfs_core.so.1 -> libzfs_core.so.1.0.0
│  ├── libzfs_core.so.1.0.0
│  ├── libzpool.so -> libzpool.so.2.0.0
│  ├── libzpool.so.2 -> libzpool.so.2.0.0
│  └── libzpool.so.2.0.0
├── newroot
├── proc
├── sbin
│  ├── mount.zfs
│  ├── zdb
│  ├── zfs
│  └── zpool
└── sys
```

重新编译Linux内核.Linux的`makefile`不能感知外部文件夹的变化,因此要删掉生成的initramfs压缩包.

```shell
cd $DISTDIR/linux
rm usr/initramfs_data.cpio* built-in.a
make ARCH=$ARCH CROSS_COMPILE=$CROSS_COMPILE -j7
```

这样,编译完的Linux内核就可以直接从UEFI Shell中启动,并启动ZFS中的系统.
如果启动失败,就会进入busybox的shell界面,可以使用`zfs`命令手动挂载根文件系统到`/newroot`,
然后手动执行`exec switch_root /newroot /sbin/init`
