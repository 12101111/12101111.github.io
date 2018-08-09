+++
title = "遇到Windows蓝屏的解决思路"
date = 2018-02-28

[taxonomies]
categories = ["电脑110"]
+++

## 什么是蓝屏

蓝屏，英语　Blue Screen of Death，缩写为：BSoD，又称为bugcheck。表现为系统突然停止响应，屏幕上出现蓝色背景的错误信息。在Windows7及之前的版本中，蓝屏为深蓝色，并含有大段英文的解决方案，告诉用户检查硬件是否正常，并建议用户卸载或移除新安装的软硬件。由于该界面过于不友好，且这段解决方案很少能解决问题，微软在Windows8及之后的版本将蓝屏改为浅蓝色背景，并显示:( 你的电脑遇到问题需要重启,以及一个没有用的二维码。

<!-- more -->

## 为什么会有蓝屏

在Windows NT操作系统中，当内核态程序发现系统无法处理的错误时，Windows会停止所有程序的运行，进入bugcheck界面，将内存中的部分数据存储到硬盘。

Windows作为微内核多任务多子系统操作系统,其内核(ntoskrnl.exe)仅负责

蓝屏发生意味着Windows NT内核本身的运行出现了问题,而且没有合适的解决办法,这与Linux的Kernel Panic,MacOS X的五国类似.这里所谓的出错指

    1.内核态程序出现意外的故障（Fault）和终止（Abort），例如编程中常见的解引用空指针等内存错误。
    2.由于安全原因主动蓝屏，例如关键的用户态进程出错会造成CRITICAL_PROCESS_DIED
    3.外部硬件故障或者不能满足Windows的运行条件

## 寻找蓝屏的原因

bugcheck执行时会将系统部分内存数据存储到`C:\windows\MEMORY.DMP`(大约1GB以上）,并将关键内存数据存储到`C:\Windows\Minidump\日期-时间.dmp`,这是分析蓝屏原因的关键.

首先打开事件查看器,查看蓝屏前有无警告或错误.

再打开[bluescreenview](http://nirsoft.net/utils/bluescreenview.zip),选择上半部分列表中最近一次的dump文件,在下半部分点击**堆栈内的地址**进行排序,红色底纹的即为崩溃时正在运行的内核态程序。可以在搜索引擎中搜索文件名+蓝屏来寻找解决方案。如果红色底纹的只有ntoskrnl.exe（Windows NT内核）就需要进一步分析。

[下载](https://www.microsoft.com/zh-cn/store/p/windbg-preview/9pgjgd53tn86)并以管理员权限打开winDbg(Windows7用户请搜索下载地址),点击左上角的file,再点击Open dump file,选择`C:\windows\MEMORY.DMP`,等待输出结果.

```output
..........................
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

Use !analyze -v to get detailed debugging information.

BugCheck 139, {3, ffff84022caef450, ffff84022caef3a8, 0}

Probably caused by : LXCORE.SYS ( LXCORE!LxpFutexEvictWaitQueue+47 )

Followup:     MachineOwner
---------

nt!KeBugCheckEx:
fffff803`cb7780e0 48894c2408      mov     qword ptr [rsp+8],rcx ss:0018:ffff8402`2caef130=0000000000000139

```

结果中的*Probably caused by : *给出了分析出的蓝屏原因.*LXCORE.SYS*为造成蓝屏的程序,*LXCORE!LxpFutexEvictWaitQueue+47*指出LXCORE程序中LxpFutexEvictWaitQueue这个函数反汇编的第47个指令是蓝屏前的最后一条指令.

点击`!analyze -v`会有更详细的说明.下面给出部分输出的介绍

```output
KERNEL_SECURITY_CHECK_FAILURE (139)
A kernel component has corrupted a critical data structure.  The corruption
could potentially allow a malicious user to gain control of this machine.
Arguments:
Arg1: 0000000000000003, A LIST_ENTRY has been corrupted (i.e. double remove).
Arg2: ffff84022caef450, Address of the trap frame for the exception that caused the bugcheck
Arg3: ffff84022caef3a8, Address of the exception record for the exception that caused the bugcheck
Arg4: 0000000000000000, Reserved
```

这里给出了蓝屏代码及产生原因,这就是蓝屏界面上显示的代码,具体的介绍在微软的[docs](https://docs.microsoft.com/zh-CN/windows-hardware/drivers/debugger/bug-check-code-reference2)上

```output
STACK_TEXT:
ffff8402`2caef128 fffff803`cb7839e9 : 00000000`00000139 00000000`00000003 ffff8402`2caef450 ffff8402`2caef3a8 : nt!KeBugCheckEx
ffff8402`2caef130 fffff803`cb783d50 : ffff8402`2caef2a8 ffff8402`2caef4b8 ffffe181`dd997080 fffff803`cb783553 : nt!KiBugCheckDispatch+0x69
ffff8402`2caef270 fffff803`cb782d37 : ffffe181`d9dd14c0 ffffd10b`19e37000 ffffd10b`19e37000 fffff809`1836bf7a : nt!KiFastFailDispatch+0xd0
ffff8402`2caef450 fffff809`182e19d7 : ffffd10b`16a80008 ffffd10b`16a80000 ffffd10b`1914a000 ffff8402`2caef7b0 : nt!KiRaiseSecurityCheckFailure+0xf7
ffff8402`2caef5e0 fffff809`18300198 : 00000000`8000000f ffff8402`2caef7b0 ffffd10b`1914a000 ffffd10b`16a80000 : LXCORE!LxpFutexEvictWaitQueue+0x47
ffff8402`2caef610 fffff809`182fc3cb : ffffd10b`16a80000 00007f01`00000000 00000000`00000000 ffffe181`dd997080 : LXCORE!LxpThreadGroupEnd+0x45c
ffff8402`2caef710 fffff809`182fddb4 : 00000000`00000011 ffff8402`2caef7b0 ffffd10b`15c9e000 ffff8402`2caef8d8 : LXCORE!LxpThreadEnd+0x207
ffff8402`2caef760 fffff809`1836bece : ffffe181`d7b79a70 ffff8402`2caef888 ffff8402`2caefa00 ffff8402`2caef8d8 : LXCORE!LxpThreadSignalApc+0xb8
ffff8402`2caef790 fffff803`cb6791c7 : ffffe181`dd997080 00000000`00e295f4 ffffd10b`15c9e000 00000000`000000ca : LXCORE!LxpNtStateThreadApcUser+0x3e
ffff8402`2caef830 fffff803`cb77b830 : 00000000`00e295f4 ffff8402`2caef8c0 ffffffff`00000000 00000000`00000000 : nt!KiDeliverApc+0x337
ffff8402`2caef8c0 fffff803`cb78379e : ffff8402`2caefa00 00000000`00000000 00000000`00000000 00000000`00000000 : nt!KiInitiateUserApc+0x70
ffff8402`2caefa00 00007f01`b70ed360 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : nt!KiSystemServiceExitPico+0x47
00007fff`dcac6190 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : 0x00007f01`b70ed360
```

前面的16进制地址不用管,最后的英文显示了调用栈的数据,即最下面的函数调用了上面的函数,顶上的函数是重启前最后执行的,显然bugcheck相关函数在上面,紧接着是造成bugcheck的函数.

搜索这些字符串可能能找到解决方案.

如果是硬件造成的bugcheck,在分析中能看到该设备的硬件ID,在设备管理器中可以确定是哪个硬件出现了问题.

## 谁导致了蓝屏

前文在内核态/用户态层面上阐述了bugcheck产生的原因，这里具体的介绍常见的原因。

### 驱动程序故障

最常见的蓝屏原因之一,尤其是显卡驱动。如果造成蓝屏的程序是`nv*.sys`或`igx*.sys`等显卡厂商的驱动程序(文件后缀有.sys、.exe、.dll)，那么可以猜测是显卡驱动程序问题。要修复驱动程序的问题，只需要进入安全模式在设备管理器卸载对应设备并勾选删除驱动程序即可。能够正常进入系统后，重新安装驱动程序，并保证安装的驱动能正常工作，有时需要安装旧版驱动。如果仍存在问题，考虑重装系统或硬件故障。

### 软件冲突

有时杀毒软件、病毒、第三方服务会导致蓝屏，安全模式下这些程序不会开机启动，可以将其卸载。腾讯的许多应用会导致此问题,尤其注意QQ Protect和TenProtect这两个腾讯的过滤驱动。

### 硬件故障

计算机的任何部件没有安装正确或有故障时都会导致使用时频繁的蓝屏或开机蓝屏，例如内存条松动老化，供电不足，主板电容损坏，硬盘损坏，显卡损坏。如果是新安装了硬件，进行过拆机，电路板存在烧焦的痕迹则可能是硬件故障，如果重装系统后依旧蓝屏，启动Linux报告硬件错误，可以推断出为硬件问题。尝试重新插拔接口或使用无水乙醇对接口进行清洗，如果问题依旧，硬件可能已经损坏。注意，机身灰尘过多，散热过差导致CPU过热也可能触发bugcheck或者直接黑屏。

### 系统文件损坏

如果在开机过程中蓝屏，甚至安全模式也无法进入，考虑重装系统。如果能进入安全模式,在管理员权限的cmd或powershell命令行中使用`SFC /scannow`或

```cmd
Dism /Online /Cleanup-Image /CheckHealth
Dism /Online /Cleanup-Image /ScanHealth
Dism /Online /Cleanup-Image /RestoreHealth
```

修复受损的系统文件.以上命令也可以离线执行,但执行时间很长且成功率很低,无特殊需求不要尝试.

尤其注意,电脑跌落、强制关机、意外断电导致的文件随机损毁会造成各种随机的故障，甚至会导致任何Windows，包括Windows PE都无法启动，卡死在LOGO或蓝屏（INACCESSIBLE_BOOT_DEVICE）。由于Windows默认挂载所有的可识别的分区并分配盘符，无法读取的硬盘和无法被NTFS.sys挂载的分区都会让Windows死机。区分是由于NTFS分区结构损坏还是机械硬盘磁头损坏/固态硬盘掉盘的可靠方法是从U盘启动Linux，Linux不会自带挂载分区，不会因某一个硬盘无法读取而卡死，如果硬盘无法读取，使用`sudo dmesg`读取内核日志，会有I/O error。这时可以格式化分区或者放弃使用这个硬盘。