+++
title = "使用Rust编写操作系统(二)"
date = 2019-09-01
[taxonomies]
categories = ["OS"]
tags = ["Rust","OS"]
+++

众所周期,BIOS作为IBM PC工程师40年前随手写下的一段汇编代码,根本不会想到能用到现在.BIOS中充满了各种各样的hack和历史糟粕.BIOS只支持从MBR分区表的第一个扇区启动,很难安装到硬盘或U盘.甚至Intel即将于2020年在新的芯片上放弃对UEFI CSM的支持,这使得未来很难直接运行只支持BIOS的系统.可惜的是,直到今日,大多数自制x86平台操作系统的资料仍旧只支持BIOS。

本文试图基于Rust操作系统开发中读者广泛的blog os系列文章提供一个能够在UEFI上运行的教学用OS.

<!-- more -->

## [BIOS是什么，UEFI又是什么](https://12101111.github.io/diannao110/firmware/%E4%B8%BB%E6%9D%BF%E5%9B%BA%E4%BB%B6.html)

老狼的文章也写的很好: [UEFI与硬件初始化](https://zhuanlan.zhihu.com/p/25941340) 和 [UEFI架构](https://zhuanlan.zhihu.com/p/25941528)

这个表列举了一些概念的演进关系:

BIOS | UEFI
:-: | :-:
实模式|保护模式
主引导记录(MBR)| ESP系统分区+GPT
引导扇区| UEFI应用
16位的BIOS中断| UEFI服务调用
DOS COMMAND.COM | UEFI shell

对于操作系统开发的初期来说,UEFI大大简化了开发流程:

1. 直接编译成EFI应用程序,几乎就是Windows PE文件,不需要研究复杂的链接脚本.(Rust编译器以原生支持x86_64_uefi的编译目标)
2. 不需要使用16位汇编编写古怪的MBR汇编代码,也不需要修改磁盘的MBR/PBR
3. 如果要上机测试，不需要进固件设置打开UEFI CSM

UEFi的服务调用使用Windows的C ABI,风格上是面向对象的,字符串使用UTF-16编码(到处充满了MS的气息).虽然Rust可用直接使用C ABI,但是满屏的`unsafe`总是很辣眼睛.Rust库[rust-osdev/uefi-rs](https://github.com/rust-osdev/uefi-rs)对UEFI的常见调用提供了一个安全的抽象。

## 一个简单的 Rust UEFI项目

首先假设你使用的是Nightly版本的编译器,如果不是,在项目根目录加入内容为`nightly`,文件名为`rust-toolchain`的文件,Cargo会在此项目中默认使用Nightly版本的编译器.

为了在Rust官方不提供预编译`stdlib`支持的平台上编译上编译Rust项目,需要安装[cargo-xbuild](https://github.com/rust-osdev/cargo-xbuild):`cargo install cargo-xbuild`

为了方便在qemu虚拟机中测试,我编写了一个工具: [bootuefi](https://github.com/12101111/bootuefi), 运行`cargo install bootuefi`即可安装.

首先新建一个binary项目,在依赖中加入

```TOML
uefi = { git =  "https://github.com/rust-osdev/uefi-rs.git" }
```

main.rs修改为：

```rust
// 此平台没有std库
#![no_std]
// 这个可执行程序不使用main作为入口点
#![no_main]
use uefi::prelude::*;
use core::fmt::Write;

// bug of compiler_builtins
#[no_mangle]
pub static _fltused: u32 = 0;

// no_mangle表示不进行符号重整，而是直接使用此符号。
// extern "C" 表示函数为C ABI的函数。
// 返回值为！表示永远不会返回
// efi_main是UEFI程序的标准入口点，参数image是UEFI程序的内存映像handle
// 参数st是UEFI系统表，是所有启动时(boottime)服务的入口.
#[no_mangle]
pub extern "C" fn efi_main(image: uefi::Handle, st: SystemTable<Boot>) -> ! {
    let stdout = st.stdout();
    stdout.clear();
    write!(stdout,"Hello, world!");
    loop{}
}

#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    loop{}
}
```

项目根目录下新建`.cargo/config`文件:

```TOML
[build]
target = "x86_64-unknown-uefi"

[target.x86_64-unknown-uefi]
runner = "bootuefi"
```

这样运行`cargo xbuild`会自动选择uefi作为目标平台，`cargo xrun`会自动使用`bootuefi`作为启动程序.

由于QEMU不自带UEFI的固件,因此需要下载`OVMF`开源固件.部分Linux软件源含有此软件,如果没有,可以从[Gerd Hoffmann's OVMF builds](https://www.kraxel.org/repos/jenkins/edk2/) 下载edk2.git-ovmf-x64*.noarch.rpm并解压得到`OVMF_CODE.fd`.默认的固件位置是项目根目录的`OVMF.fd`,可以参考`bootuefi`的[README](https://github.com/12101111/bootuefi/blob/master/README.md)修改设置.

运行`cargo xrun`,就可以在QEMU中运行这个项目了。

屏幕上应该显示出"Hello, World!".如果在符合UEFI规范的电脑上运行(关闭安全启动),应该会显示出相同的内容.

## 退出启动时环境

UEFI应用程序是为了引导系统,更新固件,硬件检测等功能的,虽然借助UEFI Shell也可以当作DOS使用,但是这个环境存在大量的限制,不是真正的操作系统:

1.不能运行驱动程序,不能修改中断描述符表,必须使用UEFI的驱动框架开发轮询式驱动.
2.不能控制内存,内存分配由UEFI固件完成
3.默认是单核单线程

而真正的操作系统,必须要退出这个环境,也就是执行`EFI_BOOT_SERVICES.ExitBootServices()`.这时候, UEFI固件停止控制硬件,UEFI系统表会从启动时表变为运行时表,大多数功能失效.UEFI运行时服务只提供:

1. 硬件时钟:读取/修改
2. 修改页表映射后重映射UEFI的运行时代码和数据
3. 修改UEFI变量
4. 重启电脑
5. 下次启动时更新UEFI固件

我们拥有了对整个电脑的控制权,同时也失去了大多数启动时便捷的服务,比如从键盘获得输入,在屏幕上显示输出,从硬盘加载文件,这意味着我们需要自己编写这些驱动.

更加可怕的是,UEFI默认使用ACPI(高级配置和电源管理接口)用于枚举设备,乃至管理多CPU核的启动,使用APIC(高级可编程中断控制器)控制CPU和南桥的中断.大多数易读的资料都基于过时的8259PIC芯片编写键盘驱动,而现代的键盘都是USB的.这意味着我们可能要在数千页的UEFI/ACPI/USB/Intel的手册中翻查我们需要的资料。

不过至少，我找到了一种在屏幕上显示的方法。

参考链接: [UEFI及ACPI标准](https://uefi.org/specifications)
