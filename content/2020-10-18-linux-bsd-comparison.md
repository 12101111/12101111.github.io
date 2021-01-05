+++
title = "庆祝OpenBSD 25周年与OpenBSD 6.8, NetBSD 9.1, FreeBSD 12.2发布, 有关Unix,BSD与Linux的看法"
date = 2020-10-18
[taxonomies]
categories = ["Linux"]
tags = ["Linux","BSD","FreeBSD"]
+++

# Linux和BSD有什么区别

## 历史

Unix源自贝尔实验室计算科学研究中心(CSRC)即贝尔实验室1127部门, Ken Thompson和Dennis Ritchie先后创造了Unix操作系统和C语言.尽管Unix最早运行在PDP-7和PDP11小型机上, 但是C语言的可移植性使得Unix移植到其他机器上也很容易.

贝尔实验室一共发布了10个版本的Unix, 也叫[Research Unix](https://en.wikipedia.org/wiki/Research_Unix), 年代从1971年11月的第一版至1989年10月的第十版.其中版本号是根据用户手册(即`man`命令)描述的.第一版到第三版是汇编写的,第4版(1973年11月)用C语言重写了,并且发表在ACM通信杂志上,第5版(1974年6月)被授权给大学等教育机构.

自此开始,AT&T发现了商机,开始销售Unix.Unix第6版(Version 6 UNIX)以$20,000的价格出售(1975年5月, 根据美元的通货膨胀,相对应现在的$95,028),这个价格非常昂贵,但是销售协议并不禁止源代码用于教学,新南威尔士大学的John Lions教授出版了Lions对Unix第六版的注释,带源代码(Lions' Commentary on UNIX 6th Edition, with Source Code), 成为贝尔实验室之外最广泛传播的Unix源代码.Unix第6版不仅销售广泛,而且一些大学和公司将Unix移植到其他机器上, 比如IBM VM/370虚拟机,VAX. 个人认为Unix分裂的源泉就是来自Unix v6的授权上, 源码是原样交付的(provide as-is), 在许多移植和开发工作发生后, AT&T并没有有效的组织起用户将修改内容提交到上游,而是自己也销售一个商业分支: PWB/UNIX.当然断人财路是AT&T这种垄断公司不能容忍的,Unix第7版(Version 7 UNIX,1979年)的协议阻止了学术用授权在课堂上使用,Lions的书也就禁售了,但是广大师生还是靠着复印继续用这本书.MIT曾经在2002年到2006年教授Unix 6th,直到后来编写了`xv6`作为Unix在x86上的一个简单的克隆.

之后的Unix第8到第10版基本上就是BSD的修改版,并包括了一些System V的功能.随后CSRC的研究重点转向了九号计划(Plan 9).到了21世纪之后, Research Unix和Plan 9的源代码陆续以自由软件协议开源了,不过也只有研究价值了.

1975年, Ken Thompson作为访问教授来到Berkeley大学,并在这里的PDP-11上安装了Unix v6, 并开始编写一个Pascal编译器.Bill Joy(后来的Sun创始人)为Unix编写了`ed`编辑器的增强版`ex`(Extend的缩写), 并改进了Pascal编译器.Bill Joy随后将整个系统和新加入的软件于1978年3月9日发布为`Berkeley Software Distribution`,即`1BSD`.Bill Joy又编写了著名的`vi`,即可视化(visual)的`ex`(ex是运行在电传打字机上的,vi是运行在视频终端上的),以及C shell(`csh`).这些改进被发布为`2BSD`.随后Berkeley购买了32位的VAX计算机,因为当时Unix的VAX移植没有利用VAX的虚拟内存功能,Berkeley的学生重写了Unix的内核,并将软件移植到了新内核上.这些被作为`3BSD`于1979年发布.`4BSD`于1980年发布,包括了`csh`的任务控制和Curses终端UI(TUI)库.Bill Joy大幅度改进了BSD内核在VAX上的性能,并作为`4.1BSD`于1981年发布.Bill Joy在1982年加入了刚成了的Sun Microsystems, 并开发了在68000上运行的BSD系统.

1978年,微软从AT&T手中购买了UNIX v7的授权,然后以XENIX的名字再卖给OEM(例如IBM和Intel).XENIX移植到了Z8001,68000以及当时流行的IBM PC.由于16位的IBM PC和其他家用计算机相较于大型机/小型机更加便宜,因此XENIX是当时销售最广的UNIX变种.然而后来微软更重视和IBM合作开发的OS/2系统,以及新的Windows NT系统,在1987年将XENIX卖给了SCO.

到了80年代, AT&T继续推进UNIX的商业化.1982年, AT&T在UNIX v7的基础上开发了UNIX System III, 并以$100(现在的$281.22)的价格出售,只能在PDP-11和VAX上运行.随后,AT&T整合了当时各个UNIX分支的功能,例如BSD的vi, 于1983年发布了UNIX System V Release1 (SVR1).然而同年,AT&T因为输了与美国司法部的反垄断官司,被拆分为7个小贝尔公司,负责各地的电话业务,以及AT&T公司从事长途电话业务和通信设备制造业务.同样是这年9月27日, 因为在大学中看不到UNIX源码的Richard Stallman决定从头开发一个**自由**的UNIX操作系统的替代品.System V的著名功能包括SysV IPC和SysV `init.d`.

由于各个UNIX差异越来越大, IEEE觉得标准化UNIX的API和命令行接口, 该标准被称为POSIX (Portable Operating System Interface).第一版于1988年发布(IEEE 1003.1-1988).

1987年,Andrew S. Tanenbaum教授为了教授操作系统课程,编写了Operating Systems: Design and Implementation教材,并在其中发行了Minix, 一个微内核操作系统. Minix 运行在16位的IBM PC/AT计算机上,可以安装到软盘,仅包括12,000行代码.尽管Minix代码可用,但是其必须和教材一起出售($69),因此不能自由分发.这限制了Minix在90年代的进一步发展.

但当时UNIX的许多重要的新功能实际上来自于BSD.尤其是1983年的`4.2BSD`包括了TCP/IP的实现.
4.3BSD(1986)改进了TCP/IP的实现,4.3BSD-Tahoe(1988)提升了代码的可移植性,并包含了著名的TCP拥塞控制算法Tahoe.由于AT&T的协议不允许源代码分发, BSD中的AT&T代码被移除, 并以BSD协议发布, 称之为Networking Release 1 (Net/1).4.3BSD-Reno(1990)则包括了对Tahoe算法的修改,Reno则成为最基本的TCP拥塞控制算法.

而AT&T于1988年发布了UNIX System V Release 4.0 (SVR4), 并整合了4.3BSD, Xenix, SunOS的功能.许多商业UNIX都迁移到SVR4上,这包括流传至今的IBM AIX, HP的HP-UX和Sun/Oracle的Solaris(即SunOS).
SVR4是AT&T发布的最后一个Unix, 1992年AT&T将Unix卖给了Novell,随后又卖给了Xenix的开发商SCO. 逐利的资本家在1996年将AT&T拆分出了朗讯,Bell实验室也被划分走.
流传至今的唯一的开源System V后代即Oracle收购Sun前的Solaris 10 (OpenSolaris),以及Fork出的illumos.

Net/1发布之后, BSD开发者 Keith Bostic(也是后来的Berkeley DB开发者)认为应该继续重写AT&T的代码, 1991年6月, Net/2发布. 一些BSD开发者成立了Berkeley Software Design Inc.(BSDi)公司继续开发Net/2并将其移植到Intel 80386,被称为BSD/OS (BSD/386).另一些BSD开发者则继续以BSD协议移植Net/2到80386, 称之为386BSD项目.

由于Net/2删掉了一些代码,因此并不完整, 1992年7月,386BSD发布可用的0.1版本.由于386BSD的开发缓慢, 一些386BSD的开发者继续开发工作,成立了FreeBSD项目.由于意见不合,另一些386BSD开发者成立了NetBSD项目.

BSDi以$995销售BSD/386的源码,远比AT&T便宜, 这招来了AT&T的官司.1993年, `4.4BSD-Encumbered`发布,并且需要Unix的商业协议.在Unix被卖给Novell后,在Berkeley大学的支持下,双方于1994年和解.1994年3月,4.4BSD-Lite发布,其进行了进一步的修改不包括非BSD的代码.BSDi和Novell的和解细节不得而知,但是公开信息表明BSDi需要将其系统迁移至`4.4BSD-Lite`.最后一个从Berkeley发布的BSD是`4.4BSD-Lite Release 2`,随后Berkeley的CSRG解散.

BSDi于2001年出售了其操作系统业务给Wind River Systems, 其服务器业务则整合为iXsystems Inc.iXsystems于2006年收购了PC-BSD项目,并开发了FreeNAS系统.2009年, 一部分FreeNAS开发者决定转移至Debian Linux并开发了OpenMediaVault.

虽然未被指控,但FreeBSD 1.0包含了4.3BSD的代码,因此1994年11月发布的FreeBSD 2.0迁移到`4.4BSD-Lite`.FreeBSD现在并不提供FreeBSD 1.0的下载.NetBSD 1.0也迁移至`4.4BSD-Lite`.1995年, 由于意见不合, NetBSD创始人之一Theo de Raadt fork了NetBSD并成立OpenBSD组织.

1991年, 大学生Linus Torvalds刚刚购买了新计算机, 具有80386CPU, 却发现只能用单用户的DOS操作系统.而当时的Minix系统为16位8088设计,没有利用上80386强大的32位处理能力以及mmu.而此时386BSD的移植工作正在进行, 并没有发布可用的版本.GNU工程的Hurd内核则开发进度缓慢,功能很不完整.剩余的商用UNIX则极其昂贵.由于没有可用的,自由的内核可以利用386的功能,Linus Torvalds按照Intel开发者手册和Operating Systems: Design and Implementation教材自己开发新的内核.开发工作是在MINIX系统上进行的,并使用了GNU工程的gcc和bash等软件.Torvalds于1991年8月宣布了开发这个内核的消息,并于10月发布0.0.1版.第一个版本功能很不完整,包括了6744行C, 2143行C头文件, 1770行汇编和459行makefile.这个内核用于386 AT计算机,需要有VGA显示适配器和IDE硬盘, 此外键盘只支持芬兰键盘,美式键盘会输出错误的字符.Linux 0.0.1使用Minix的文件系统, 可以读取IDE硬盘中的分区,内核可以安装到软盘上.这一年,Linux随后发布了0.0.2,0.03,0.10,0.11, 然而0.0.2和0.0.3的代码现已经丢失. 1992年 1月,Torvalds发布了Linux 0.12, 支持了交换分区, SVGA显卡, Job Control, 虚拟终端, 387浮点协处理器模拟,符号连接, 此时的Linux内核功能已较为完善.下一个版本版本号直接跳到0.95, 然而Torvalds发现了一些bug,因此版本号增加的速度降了下来.0.96成功运行了X服务器,0.97包含了Linux第一个原生文件系统ext, 0.98 包括了TCP/IP网络支持.到1993年34月,第一个BSD在386平台的移植NetBSD 0.8发布时(1993年4月19日),经过两年的开发,Linux 0.99.8 (1993年4月9日)已经具有9万2千行代码,包含了一些SCSI硬盘驱动, 支持多种文件系统,支持网络,支持鼠标键盘和X,支持内核压缩存储的成熟内核, 而这一版的NetBSD还包括具有版权问题的文件.而没有版权问题的BSD, NetBSD 1.0于1994年10月发布,FreeBSD 2.0于1994年11月发布, 而Linux 1.0已经于半年前的1994年3月14日发布了,包含了17万6千行代码.

历史没有如果,但是Linus Torvalds本人也表示,如果1991年的时候386BSD已经发布,Linus Torvalds将不会开发Linux, 但BSD由于慢了2年,丢掉了自己的市场.

## Kernel vs OS

不同于BSD, Linux只是一个内核,而BSD是一个系统.Linux发行版则是GNU/Linux特有的概念,其维护了系统可以安装的软件.1993年成立的Debian项目和Redhat公司至今具有巨大的影响力.

Linux发行版和BSD系统的区别就体现在开发和发行这个问题上.Linux发行版将所有的软件,包括内核, libc, shell, Xorg, 浏览器, 游戏都作为软件包,由包管理器管理(apt/dpkg,rpm/yum/dnf,pacman,emerge),而Linux发行版的开发者(打包员)编写打包脚本(control/rules, spec, PKGBUILD, ebuild),然后通过构建工具编译软件包,经过测试后上传到软件源.

而BSD首先将软件分为base系统和ports.base系统包括了Unix规范中包括的所有软件,例如内核,shell,编辑器vi,编译器cc,以及这些软件需要的库.base系统中的内核,软件,库都放在同一个源代码管理仓库中.虽然这些软件有很多是来自第三方的,例如几乎所有的BSD都使用Clang和llvm作为工具链,但是llvm的源码被vendor进内核的仓库中.而ports则是所有不包括在base系统中的软件,和Linux发行版一样,具有包管理器(pkg,pkgsrc),而打包脚本的采用BSD make.用户既可以使用包管理器下载二进制软件,也可以直接使用BSD make在对应的目录下运行`make && make install`从源码安装软件(就像Archlinux的AUR和Gentoo Linux一样).

另一个区别就是BSD遵循传统,`/`(单用户模式所需文件),`/usr`(多用户模式所需文件),`/usr/X11R6`(X窗口系统),`/usr/local`(Ports安装的软件和配置文件)具有明确的作用,而Linux所谓的usr merge将`/`和`/usr`合并(因为Systemd并没有单用户模式,且几乎无法不依赖/usr),`/usr/local`仅用于用户手动编译安装的软件.

BSD对base系统保证其ABI的稳定性,即升级base系统的安全补丁不会导致现有Ports软件需要重新编译或者升级,这同样包括内核ABI的稳定性.而Linux发行版对ABI稳定性则有不同的策略.例如红帽的RHEL保证了内核和系统的ABI稳定性,而ArchLinux则不承诺任何ABI稳定性,甚至阻止部分升级,Gentoo Linux的软件均为手动编译,因此ABI变动时需要重新编译软件包.另一方面,Linux内核的syscall总是稳定的,即升级Linux内核不应该破坏任何软件(除非该软件依赖了不稳定的部分,例如sysfs的输出),但Linux故意的不稳定API和ABI,导致Linux内核模块只能被对应的*那一个*Linux内核加载.而BSD的内核模块可以被相同大版本号的内核加载(不包括不支持内核模块的OpenBSD).另一方面,当大版本号发生变动时,BSD则不会做出任何ABI稳定性的承诺,因此要小心的从FreeBSD 10.2 升级到 11.1.

| 项目          | Linux                                                | BSD                     |
| ------------- | ---------------------------------------------------- | ----------------------- |
| 协议          | GPLv2                                                | BSD                     |
| 内核          | 独立开发                                             | 随系统一起开发          |
| 基本库/命令行 | GNU等组织开发,发行版发行                             | 随系统一起开发发行      |
| 包管理器      | 发行版开发并发布软件源                               | 系统更新,Ports,二进制源 |
| 稳定性        | 内核syscall对用户态稳定,ABI不稳定,用户态随发行版策略 | base系统大版本内ABI稳定 |

## 功能

### drm

drm是Linux的显示驱动框架,同等概念的还有Windows NT6/10的 WDDM. 近些年X.Org和FreeDesktop.org虽然开发了大量供Unix-like操作系统的桌面软件,但是很多都是Linux专用的.例如Mesa 3D和Xorg使用DRI来和显卡驱动交互,但DRI只能和drm交互,这导致BSD无法使用相关的软件.BSD给出的解决方案是,将Linux的drm模块移植到BSD中,并使用Linux内核兼容接口(例如FreeBSD的linuxkpi)使用drm.虽然amdgpu和i915都是以MIT协议发布drm中的驱动,但是drm本身是GPLv2授权的,因此BSD的移植也是GPLv2授权的.由于是移植自Linux, 因此BSD的显卡驱动支持总是慢于Linux, 并且具有一些Linux没有的bug.

### 文件系统

Linux支持ExtFS系列, ReiserFS, 来自AIX的JFS, 来自SGI IRIX的XFS, ~~永远不稳定的~~BtrFS,三星给手机开发的F2FS,以及许多小众的文件系统(例如Minixfs), 不支持POSIX的vFAT和来自三星的代码微软的专利的ExtFAT,以及支持用户空间文件系统的FUSE框架.

Ext系列是Linux长期使用的文件系统格式,但是随着ext4的推出,ext系列的可扩展性越来越差(虽然其名称为extended filesystem).2007年, Btrfs原作者, Chris Mason加入当时还没有收购Sun的Oracle公司, 号召社区开发一个新的Linux原生的Cow文件系统.2009年, Btrfs进入Linux主线.Btrfs支持许多有用的文件系统功能,[其wiki上对功能列表进行了记载](https://btrfs.wiki.kernel.org/index.php/Main_Page#Features),大部分功能都是稳定的,有一些功能,例如原生RAID5,具有一些bug,不应该日常使用,[其wiki也记载了功能是否稳定](https://btrfs.wiki.kernel.org/index.php/Status).然而经过10年的开发后,仍有人在升级内核版本后或其他操作后,btrfs发生文件丢失,文件损坏或内核panic.当然这都是小概率事件,BtrFS广泛的在企业环境中使用,因此如果不使用不稳定的功能,通常不会发生文件损坏的事故.

长期以来Unix则使用Unix file system (UFS), 其历史最早追溯到version 7 Unix,但是广泛使用的版本是4.2BSD以及SVR4.由于BSD对该文件系统的代码做出了很多优化,有些系统也将UFS称为Berkeley Fast File System或BSD Fast File System (FFS).现代Unix大多衍生自4.2/4.4BSD-Lite和SVR4, 因此许多Unix都支持UFS/FFS, 例如Solaris OS, HP-UX, 以及早期的NeXTStep和Mac OS X, 以及衍生自4.2BSD和4.4BSD-lite的386BSD的诸多后代: FreeBSD, OpenBSD, NetBSD, DragonFly BSD.然而随着各个分支的演化, UFS各个实现具有不同的功能,例如FreeBSD 5.x之后包括了UFS 2, NetBSD的FFS从FreeBSD移植了UFS2的功能(NetBSD 5.0), 而直到OpenBSD 6.7发布(2020年4月), OpenBSD才默认使用FFS2.另一方面,FreeBSD的UFS2, NetBSD和Solaris日志使用不同的实现.Linux包含一个UFS的只读驱动,且在挂载时需要指定BSD的变种.

UFS代码虽然久经考验十分稳定,但是功能很少,正如ext4一样.Sun公司开发的Solaris OS也在使用UFS,而为了摆脱UFS的限制,Solaris的开发者Jeff Bonwick,Bill Moore和Matthew Ahrens开始开发Solaris的下一代文件系统和存储管理方案,也就是Z filesystem.Solaris当时作为商业UNIX是一个闭源项目,但是Sun为了吸引用户和开发者不加入Linux阵营, 2005年Sun启动了OpenSolaris项目并以CDDL协议开源了Solaris 10系统的源代码,这也包括正在开发的ZFS代码.2006年,ZFS稳定版作为Solaris 10 U2更新发布. 2006年,ZFS通过FUSE移植到了Linux上.2007年,Apple将ZFS移植到MacOS X上,但Apple于2009年放弃了该项目, MacZFS项目继续该代码的开发.2007年,ZFS移植到FreeBSD并于2008年作为FreeBSD 7的一部分发布.2007年 ZFS也移植到了NetBSD,并于2009年发布.2008年, 原生版本的ZFS到Linux的移植开始开发,被称为ZFS on Linux项目.

有关ZFS的实现细节和与Btrfs的对比可以参见fc老师(farseerfc)的两篇很棒的文章: [ZFS 分层架构设计](https://farseerfc.me/zhs/zfs-layered-architecture-design.html) [Btrfs vs ZFS 实现 snapshot 的差异](https://farseerfc.me/zhs/btrfs-vs-zfs-difference-in-implementing-snapshots.html).

而DragonFly BSD则开发了HAMMER文件系统,使用B+树,支持

另一方面,Apple Darwin(包括macOS, iOS, watchOS和tvOS)支持HFS,HFS+和APFS.HFS是一个传统的基于B树的文件系统,HFS+支持了Unicode,可选的日志和大小写敏感功能,ALC,透明压缩(基于Deflate算法),加密功能.令人沮丧的是,HFS+并不支持数据校验!!!Apple意识到HFS+并不是一个21世纪的文件系统,因此设计了APFS(Apple file system),并于2017年发布在iOS 10.3和macOS 10.13 High Sierra上.APFS基于Cow的B树, 面向闪存优化,支持原生全盘加密, 文件克隆(reflink), 瞬时快照,元数据校验和, 透明压缩, 子卷(Volume)空间共享, 基于Cow的数据一致性保护.

既然提到Cow文件系统, Windows server 2012也包括了一个新的文件系统Resilient File System (ReFS), 基于B+树和Cow, 然而Windows不能启动到Refs,且经常随win10的更新发生break change.由于微软并没有公开其资料,因此并没有第三方工具或系统可以挂载Refs.Refs也不支持快照,透明压缩等其他Cow文件系统常见的功能,因此是4种Cow文件系统中最差的一个.

除了FreeBSD和illumos对ZFS具有完整的支持,包括从ZFS启动, NetBSD长期依赖使用一个较老的illumos上的ZFS移植,并于2020年转向FreeBSD的ZFS移植,但仍然不支持从ZFS启动.而OpenBSD认为CDDL过于严格,并不支持ZFS.4大BSD均通过ext2fs驱动支持Linux的ext2-ext4文件系统.

#### ZFS的争议

然而由于商业上的失败,Sun Microsystems于2010年被Oracle收购, Oracle随即终止了OpenSolaris项目,次年发布的Solaris 11成为闭源软件.对此很不满的社区fork了OpenSolaris, 成立illumos继续开发OpenSolaris, 这也包括ZFS. 2013年3月, ZFS on Linux发布首个稳定版, 同时MacZFS项目宣布使用ZFS on Linux的代码.随后的2013年9月,OpenZFS项目成立,致力于整合illumos, Linux, FreeBSD和macOS上的开源ZFS实现, Matt Ahrens等开发者继续独立于Oracle开发开源ZFS. 2020年, 基于ZFS on Linux代码的OpenZFS 2.0发布, ZFS on Linux和FreeBSD 13.0首次采用了同一份代码.

然而Sun当初开源OpenSolaris*可能*有意的选择Common Development and Distribution License (CDDL)协议基于Mozilla public license 1.1,而FSF在2005年发表声明,CDDL与GPL不兼容,并不推荐使用CDDL协议.

根据CDDL 3.1款:

- 您以"可执行文件"形式分发,或以其他方式提供"所涵盖的软件",您也必须以"源代码"形式提供，并且该"源代码"形式必须仅根据本许可条款(CDDL)进行分发.
- 您必须在分发或以其他方式提供的"所涵盖软件"的"源代码"形式的每份副本中包含此许可(CDDL)的副本.
- 您必须通知所有收到以"可执行文件"形式分发的"所涵盖的软件"的收件人, 告知他们如何以合理的方式或习惯用于软件交换的媒介获得"所涵盖的软件"的"源代码".

注意到CDDL协议"所覆盖的软件"分发时也必须使用CDDL协议提供源代码.

CDDL 3.5款:

- 您可以根据本许可的条款或您选择的许可的条款分发"所涵盖的软件"的"可执行文件"，其中可能包含与本许可不同的条款，前提是您遵守本许可的条款并且"可执行文件"的许可证不会试图限制"本许可证"规定的权利或更改"源代码"形式中的收件人权利.
- 如果您以不同的许可证以"可执行文件"分发涵盖软件，则必须绝对清楚地指出，与本许可证不同的任何条款仅由您提供，而不是由初始开发者或贡献者提供。 您在此同意赔偿初始开发者和每位贡献者因您提供的任何此类条款而引起的初始开发者或该贡献者的任何责任。

所以CDDL允许二进制文件以其他条款分发,但收件人在CDDL3.1款规定的以CDDL协议提供源代码的权利不能被剥夺.

而GPL v2 2.b款:

```txt
You must cause any work that you distribute or publish, that in
whole or in part contains or is derived from the Program or any
part thereof, to be licensed as a whole at no charge to all third
parties under the terms of this License.
```

您必须使您分发或发布的全部或部分包含本程序或其任何部分的内容或源于本程序或其任何部分的任何"作品"不得收费的以本协议授权给任何第三方.
这也就是著名的GPL感染, 任何"作品", 如果是派生自GPLv2的程序,或者包含GPLv2的程序的部分或全部,那么这个程序就被感染了,该程序的全部代码就需要以GPLv2的协议发布.

值得注意的是,GPL不限制任何不进行再分发的修改.例如修改某GPL授权的服务器程序,然后运行该程序为他人提供服务.由于该服务器程序本身没有分发,因此不需要公开其源代码.此外在本地混合Linux和ZFS的代码,并编译出内核也不违反任何协议,但该内核不能分发给任何人.

但是,对作品,尤其是派生作品的定义在各国的版权法中定义各不相同.因此将原本是属于Solaris的ZFS和Linux编译出一个内核,是否仍是派生作品,则充满了争议.
Linus Torvalds多次声明, Linux内核的GPL例外只适用于和内核交互的用户空间程序, 任何链接到内核的模块均被GPL协议覆盖.
但是NVIDIA的闭源显卡驱动则通过添加LGPL的兼容层隔离了闭源的显卡驱动,避免其闭源软件被GPL感染.Linus Torvalds曾多次反对该行为,包括公开发表NVIDIA是最烂的公司, "So, nvidia fxxk you".
但是,没有针对NVIDIA该行为的法律行动.

由于内部律师评估后认为发布ZFS on Linux的二进制文件没有法律风险, 2016年, Canonical决定在Ubuntu 16.04中发布预编译的zfs on linux内核模块. 按照FSF的理论,这直接违反了GPL的协议, FSF主席Richard Stallman发表了[声明](https://www.fsf.org/licensing/zfs-and-linux), 认为Linux和ZFS代码链接成二进制文件时,最终的程序(Linux内核)构成了"作品",无论该链接是镜头链接还是动态链接(即内核模块),FSF认为该问题的最终的解决方案是ZFS的作者以GPL兼容的协议重新发布ZFS的源代码.FSF也表示如果Linux版权所有者不追责, 不代表GPL不起作用, 同时敦促Linux应该升级到GPL v3(Linus Torvalds多次表示Linux不会使用GPL v3).

2019年, Linux内核维护者和最活跃的开发者Greg Kroah-Hartman提交了一个commit, 阻止非GPL代码使用FPU的函数(通常内核不允许使用FPU,因为FPU的寄存器数量多大小大,要保存FPU的寄存器的值到内存才可以使用FPU同时不影响用户态程序,在使用完FPU后还需要手动恢复FPU寄存器的值,而该保存和恢复需要手动调用kernel_fpu_begin和kernel_fpu_end函数).ZFS使用了SSE/AVX来加速允许,而因为无法访问这两个函数,ZFS只能不使用FPU,这导致性能下降. 2020年, Linus Torvalds在论坛上被问到[为何Linux破坏了ZFS的功能](https://www.realworldtech.com/forum/?threadid=189711&curpostid=189841), Linus表示他无法维护任何外部的内核模块, 也不能将ZFS提交到Linux里, "在我收到甲骨文的主要法律顾问或最好由拉里·埃里森本人签署的甲骨文正式信件之前，我无法合并ZFS的任何工作. 其他人认为将ZFS代码合并到内核中是可以的，使用模块接口也可以，这是他们的决定。但是考虑到Oracle的诉讼性质以及有关许可的问题，我永远都不会感到安全.而且我对某些'ZFS填充层'东西完全不感兴趣，有些人似乎认为这会隔离两个项目。这对我们这边没有任何价值，而且考虑到Oracle的接口版权诉讼（指Oracle指控Google的Android复制了Java的API），我不认为这是对的。" 事实上, Oracle并不关心来自Sun的Solaris的未来, 而是关心自己的Oracle Linux,和诞生于自己的Btrfs. Oracle也已经把另一个来自Solaris的CDDL授权的Dtrace以GPL授权重新发布, 但是没有重新发布ZFS,可能就是因为~~烂泥扶不上墙的~~Btrfs.

### Firewall

OpenBSD的开发者喜欢造轮子,尤其是因为原软件的协议不符合他们的心理预期.IPFilter曾经是一个适用于Unix的开源防火墙,但是OpenBSD认为其协议有问题, 开发了自己的版本: Packet Filter (PF, pf). 随后pf被移植到FreeBSD, Apple Darwin, NetBSD, DragonFly BSD, Solaris, QNX. pfSense和OPNsense是基于FreeBSD和pf的开源路由器/防火墙操作系统.

Linux有和pf相似功能的netfilter/iptables.说到底netfilter和pf的功能差不多, 尤其在各大云厂商的疯狂优化下, 二者性能都得到了极致的优化.

值得注意的是netflix是FreeBSD和pf的一个重要的使用者.

### BPF

Berkeley Packet Filter, 伯克利包过滤器, 简称BPF, 是一种Unix内核上的用于处理网络数据包的寄存器虚拟机, 因为工作在内核态, 避免了用户态程序频繁的复制, 因此性能很高.Linux和BSD都支持BPF.

Linux基于BPF的思想开发了eBPF(extended BPF), 这种虚拟机不仅可以拦截网络包, 还可以hook各种内核子系统, 追踪其运行轨迹.LLVM和gcc还支持使用高级语言编写eBPF的程序.

### Kqueue

传统Unix 拥有 select(2)和poll(2)系统调用用于监听文件描述符, 但是针对大量文件时poll的效率很低. Linux因此加入了改进的epoll系统调用, 可以监听一系列事件, 并在时间触发时得到通知. kqueue是FreeBSD内核的一个事件队列机制, 不仅可以处理文件描述符,也可以处理信号, 异步IO,子进程等事件. kqueue随后被移植到netbsd, openbsd, dragonfly bsd和macOS.长期以来, Linux epoll的设计被认为不如BSD的kqueue和Windows/AIX/Solaris的IOCP, 直到类似功能的io_uring的出现.

### jail

jail是一个高级的chroot, 隔离了文件系统, 网络, pid 和uid, 功能类似Linux的cgroup, 但是出现的更早.然而基于cgroup的docker生态远好于jail.

### bhyve

bhyve是FreeBSD的hypervisor和虚拟机管理器, 类似qemu/KVM