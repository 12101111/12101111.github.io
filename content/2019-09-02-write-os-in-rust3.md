+++
title = "使用Rust编写操作系统(三)"
date = 2019-09-02
[taxonomies]
categories = ["OS"]
tags = ["Rust","OS"]
+++

本文主要解释PC平台的图形显示标准.

<!-- more -->

## VGA与GOP

在古老的IBM XT上，显示器只有简单又复杂的工作:显示文字，古老的MDA(单色显示适配器)和昂贵的CGA(彩色显示适配器)都只能显示文字,只不过后者还能改变文字的颜色.这就是著名的文本模式，至今的显卡仍可以模拟此模式。

MDA可以显示80列25行字符，每个字符是分辨率为9x11像素的位图，但包括空白后的大小为9x14。因此有效的分辨率为720x350。CGA则可以显示320×200,4种颜色(从16色的硬件调色板选择)/640×200,黑白,也可以支持文本模式:80×25，8×8像素字体.由于CGA文字的分辨率低,因此MDA更加流行,直到第三方的大力士显卡可以同时支持"高清"的MDA字体和720x348的图形模式。随后显存越来越大，第三方显卡可以显示更大的分辨率。

在字符模式下,由编码生成画面是由显卡完成的。每一张显卡内都具有能容纳256个字符字体的ROM,西方不同地区的字母具有不同的数量，MS-DOS和IBM引入了代码页的概念来区分不同的编码，这一概念延续到当今的Windows上。

常见的美国地区的代码页是437，字符为：

![codepage 437](https://upload.wikimedia.org/wikipedia/commons/f/f8/Codepage-437.png)

除了ASCII外,ROM中还包含了一些用于模拟图标的线和一些数学符合和图标。

1987年，IBM在PS/2系列计算机中引入VGA(视频图形阵列)标准,支持640x480/16色,320x200/256色显示，标准还包括沿用至今的15针VGA连接器。这也是当今PC显卡遵守的最后一代IBM制定的标准，后来IBM对PC市场的控制力大大降低，NEC联合8家显卡制造商于1989年成立VESA(视频电子标准协会),制定了800x600/16色的SVGA标准,并随后迅速的扩展颜色数和分辨率.

在较老的系统,比如DOS/Windows3.x中,进入图形模式需要使用16位的BIOS中断指令:`int 10h`,VESA标准还包括了VESA BIOS扩展(VBE),允许通过BIOS中断调整到高于VGA标准的分辨率.BIOS中断需要实模式或者虚拟8086模式,因此无法在64位的长模式下使用.

大多数操作系统处理显示的方法为:

1. 实模式下的bootloader调用VBE调整显示分辨率和色深,VBE会在内存空间中创造出一块指向DRAM(核显)或者PCI(独显)的帧缓存.
2. bootloader加载系统内核并进入保护模式/长模式
3. 内核加载专门编写的显卡驱动程序,以使用2D/3D加速功能

可以预见的是，大多数个人开发的操作系统内核，如果没有大量的资金、人手和精力，单独针对每一款硬件开发、逆向工程驱动程序简直是天方夜谭。使用VBE是比较合适的选择。

而到了UEFI的时代，BIOS中断无法使用。UEFI作为一个标准化的固件规范，提出了一种硬件无关的协议：GOP（图形输出协议）.GOP支持以下特性:

1. 平台无关,不仅限于x86架构，还支持arm和RISC-V等,不需要显卡继续兼容古老的MDA文本模式和VGA,因此也不需要主板和显卡附带16位x86代码(VBIOS ROM).
2. 可扩展任意的分辨率,不仅限于VGA/VESA固定的分辨率.同时可以枚举/设置分辨率.
3. 支持多GPU和多显示器同时工作.
4. 具有有限的2D加速(Bitblt)功能

GOP的目标是在系统启动时显示LOGO,本地化的文字,显示图形化的配置屏幕,以及在操作系统没有加载高性能显卡驱动时显示图形.很可惜的是,2D加速功能在退出启动服务后就不能使用了,操作系统只能使用GOP服务创建好的帧缓冲(framebuffer),也不能调整分辨率。
