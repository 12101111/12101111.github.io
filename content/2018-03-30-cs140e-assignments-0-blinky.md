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

# 0x2:控制LED小灯

按照第一张图连好,灯应该会亮,再将`files/gpio16-blink.bin`重命名为`kernel8.img`放到tf卡根目录,将连线改为第二张图,小灯应该会闪烁.接下来两部我们要自己写出`gpio16-blink.bin`这个程序.

# 0x3:C

本节要用C语言写出gpio16-blink.bin.首先要安装编译器,安装指示做就行了.注意修改完环境变量后需要重启终端或重新登陆.

...