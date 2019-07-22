+++
title = "使用Rust编写操作系统(二)"
date = 2019-03-29
draft = true

[taxonomies]
categories = ["OS"]
tags = ["Rust","OS"]
+++

Rust语言编写的教学用内核也不少了,但是可惜的是,缺少一个能够方便的实机运行的系统或引导器.

中所周期,BIOS作为IBM PC工程师40年前随手写下的一段汇编代码,根本不会想到能用到现在.BIOS中充满了各种各样的hack和历史糟粕.BIOS只支持从MBR分区表的第一个扇区启动,很难安装到硬盘或U盘.甚至Intel即将于2020年在新的芯片上放弃对UEFI CSM的支持,这使得未来很难直接运行这些只支持BIOS的系统.作为一个面向未来的教学用操作系统,应当有能力在真实的硬件上运行,而不是一辈子在虚拟机或模拟器中.

现代的x86计算机使用UEFI引导,而最简单易懂blog os并没有提供对UEFI的支持,仅支持传统的BIOS.blog os目前的架构是一个MBR上的bootloader加上一个elf格式的内核镜像.作者希望对UEFI的支持方式是另一个UEFI application作为bootloader.

不过,我的态度是内核直接作为UEFI application启动,并在合适的初始化后退出UEFI的启动服务.

本文试图基于blog os提供一个能够在UEFI上运行的教学用OS.

<!-- more -->

# [BIOS是什么，UEFI又是什么](https://12101111.github.io/diannao110/firmware/%E4%B8%BB%E6%9D%BF%E5%9B%BA%E4%BB%B6.html)
