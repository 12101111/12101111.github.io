+++
title = "cs140e-assignments-0-blinky"
date = 2018-03-30
draft = true
[taxonomies]
categories = ["Rust"]
tags = ["Rust", "RaspberryPI", "OS", "cs140e"]
+++
# 介绍

cs140e的第0个实验主要是让我们搭建好实验环境, 因此内容非常简单.

实验地址:https://web.stanford.edu/class/cs140e/assignments/0-blinky/
<!-- more -->
# 0x0:准备

我们需要安装好Linux,MacOS,或者在Windows10上安装Linux子系统(WSL). 总之,整个课程需要64位的unix环境.

其次,我们要购买树莓派3、面包板、杜邦线（母对母、母对公、公对公若干）、CP2102USB转TTL转换器、TF卡(推荐16g)、TF转SD卡套或USB读卡器、彩色发光二极管一套、常用电阻一套.这些东西缺一不可,淘宝上均有出售.

然后,安装git, wget, tar, screen, make这些程序,并执行

```
git clone https://web.stanford.edu/class/cs140e/assignments/0-blinky/skeleton.git 0-blinky
cd assignment0
make fetch
```

本次实验的代码就下载好了.

# 0x1:点亮树莓派

一般Linux和Windows不需要安装驱动就能识别到CP2102,插入CP2102后,Linux下出现`/dev/ttyUSB0`(0可能是其他数字)设备文件即可.如果是Windows10系统,右击开始按钮,打开设备管理器,在`端口(COM和LPT)`中会出现类似`Silicon Labs CP210x USB to UART Bridge (COM3)`的设备,如果图标带有感叹号,右击选择更新驱动程序,选择`自动搜索更新的驱动程序软件`,如果不行,在网上或买家那里找一个能工作的驱动.随后打开WSL,设备文件应该是`/dev/ttyS3`(数字应该对应设备管理器中的COM号)

随后按照实验要求连接好,其中CP2102的AS+可以不连接,CP2101的GND可以连接任意一个树莓派的GND.如果连接了CP2102的AS+,那么树莓派的供电将会来源于电脑的USB口,但是大多数笔记本的USB供电不足.如果不连接AS+,那么开机时只需要插入microUSB电源就行了.注意,千万不要把GPIO中的5v PWR直接和GND相连,可能会烧毁板子.

把tf卡插到电脑上,格式化成FAT32,将`files/firmware`中的文件复制到分区根目录,再将`files/activity-led-blink.bin`这个文件复制到分区根目录并重命名为`kernel8.img`.然后拔出tf卡,插进树莓派,将CP2102插到电脑上,接通电源(如果需要的话),树莓派上microUSB处有一个绿灯在闪,同时CP2102上的灯也会闪,这代表有数据正在通过串口传输,执行 `sudo screen /dev/<your-path> 115200` 就能看到传输的数据,"On!"和"Off!"伴随着灯的开关出现,按下CTRL+A,再按下k,再按下y来退出.

注意,如果你正在使用树莓派3B+,你需要下载Raspbian Stretch Lite并从中提取FAT分区的文件,并替换`files/firmware`中的二进制固件`start.elf`.

# 0x2:控制LED小灯

按照第一张图连好(一定要使用合适的电阻),灯应该会亮.再将`files/gpio16-blink.bin`重命名为`kernel8.img`放到tf卡根目录,将连线改为第二张图,小灯应该会闪烁.接下来两部我们要自己写出`gpio16-blink.bin`这个程序.

# 0x3:C

本节要用C语言写出`gpio16-blink.bin`.首先要安装编译器,安装指示做就行了.注意修改完环境变量后需要重启终端或重新登陆.

在大多数现代Soc上,与外部设备的IO是直接通过读写设备映射在内存空间的特殊地址来进行的,与内存不同,有些地址只读,有些地址只写.

我们需要使用C语言对`GPFSEL1`写入数据来使`Pin16`作为输出,`GPFSEL1`是可读写的,根据92页的表格,对`FSEL16`也就是`GPFSEL1`的18-20位写入001即可.为了不影响其他pin,这里用两次操作来完成.

```c
 *GPIO_FSEL1&=~(0x11<<19);
 *GPIO_FSEL1|=0x1<<18;
```

随后交替对`GPSET0`和`GPCLR0`第16位写入1即可,由于对这些位写0没有影响,同时这些地址不可读,因此直接对地址赋值.

```c
  for(;;){
  	*GPIO_SET0=0x1<<16;
  	spin_sleep_ms(250);
  	*GPIO_CLR0=0x1<<16;
  	spin_sleep_ms(250);
  }
```

运行`make`,将`blinky.bin`保存到SD卡重命名为`kernel8.img`,上电,led开始闪烁.

# 0x4:Rust

安装Rust,Xargo

```bash
curl https://sh.rustup.rs -sSf | sh
rustup default nightly
rustup component add rust-src
cargo install xargo
```

同时对文件进行修改以适应最新版的Rust,并修正一个bug.

```diff
diff --git a/phase4/src/lang_items.rs b/phase4/src/lang_items.rs
index 134f657..89b3467 100644
--- a/phase4/src/lang_items.rs
+++ b/phase4/src/lang_items.rs
@@ -1,7 +1,11 @@
 #[lang = "eh_personality"] pub extern fn eh_personality() {}
+use core::intrinsics;
+use core::panic::PanicInfo;
 
-#[lang = "panic_fmt"] #[no_mangle] pub extern fn panic_fmt() -> ! { loop{} }
-
+#[panic_implementation]
+fn panic(_info: &PanicInfo) -> ! {
+    unsafe { intrinsics::abort() }
+}
 #[no_mangle]
 pub unsafe extern fn memcpy(dest: *mut u8, src: *const u8,
                             n: usize) -> *mut u8 {
diff --git a/phase4/src/lib.rs b/phase4/src/lib.rs
index 8019dbf..470c1aa 100644
--- a/phase4/src/lib.rs
+++ b/phase4/src/lib.rs
@@ -2,8 +2,8 @@
 #![no_builtins]
 #![no_std]
 
-extern crate compiler_builtins;
-
+#![feature(panic_implementation)]
+#![feature(core_intrinsics)]
 pub mod lang_items;
 
 const GPIO_BASE: usize = 0x3F000000 + 0x200000;
@@ -14,7 +14,7 @@ const GPIO_CLR0: *mut u32 = (GPIO_BASE + 0x28) as *mut u32;
 
 #[inline(never)]
 fn spin_sleep_ms(ms: usize) {
-    for _ in 0..(ms * 600) {
+    for _ in 0..(ms * 6000) {
         unsafe { asm!("nop" :::: "volatile"); }
     }
 }
 ```

 随后照葫芦画瓢就行了.

 # 参考

 如果我写的太差,你可以看一看知乎上两位上海交大的大佬写的.

 [https://zhuanlan.zhihu.com/p/38674307](https://zhuanlan.zhihu.com/p/38674307)
 [https://zhuanlan.zhihu.com/p/33115552](https://zhuanlan.zhihu.com/p/33115552)