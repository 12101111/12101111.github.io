+++
title = "Xbox one 的安全措施"
date = 2021-05-11
[taxonomies]
categories = ["OS"]
tags = ["Xbox","OS", "Windows"]
+++
本文将翻译Xbox Live团队Xbox One安全系统架构师Tony Chen在2019年平台安全峰会(Platform Security Summit 2019, PSEC 2019)上的演讲

PSEC 2019官方页面, 内附演讲视频: [抵御物理攻击, Xbox one的故事(Guarding Against Physical Attacks: The Xbox One Story)](https://www.platformsecuritysummit.com/2019/speaker/chen/)

<!-- more -->

# [Tony Chen](https://www.microsoft.com/en-us/research/people/tonychen/)是谁

根据其在微软网站上的介绍, 他目前是微软核心操作系统团队软件工程师和安全架构师.

他是Xbox Live团队自2000年起的创始成员. 自2000年到2010年, 他作为Xbox Live的开发者, 领导, 开发主管和架构师. 在2011年到2013年, 他作为开发主管负责Xbox one安全, 并与Xbox硬件团队和AMD成功开发了Xbox one主机并于2013年发布, 且至今没有被破解, 盗版或开挂.

2014年后, 他在微软核心操作系统团队工作, 并通过软硬件改进的方式改进Windows设备的安全性, 在最近几年, 他在探索用于克服C/C++代码的内存安全性问题的指令集架构.

# Xbox 软硬件简介

这里为了方便正文的理解, 先介绍一下xosft.dev上记载的Xbox的一些软硬件细节.

## EMMC

Xbox主板上有一个8GB的EMMC 4.5 NAND Flash, 用于存储操作系统文件, 因此即使硬盘坏掉, 系统也可以启动到恢复模式.

Team Xecutor早在2013年找到了一个电路hack的方法把这个NAND接到一个SD卡上, dump出了其中的内容, [方法见该文章](https://www.xpgamesaves.com/threads/how-to-read-write-xbox-one-nand-filesystem.95025/)

其中的内容被称为[Xbox Boot file system](https://xosft.dev/wiki/xbox-boot-file-system/), XBFS包括了bootloader, 配置文件, 密钥, 以及系统固件, 但大多数内容被加密了.

## [Xbox Operating System](https://xosft.dev/wiki/xbox-operating-system/)

PS4使用了FreeBSD 9, Switch魔改了曾用于3DS的Horizon OS 微内核操作系统, 混合了NVIDIA的Android for Tegra的图形组件(比如SurfaceFlinger), 而Xbox one既然是微软出的, 那肯定用的是Windows.

Xbox one有3个Windows OS运行在主机上, 分别是HostOS, SystemOS, GameOS

### HostOS

HostOS是基于Windows 8修改而来的专用系统, 通过基于Hyper-V的虚拟机平台管理SRA和ERA系统. HostOS也包括了所有硬件的驱动程序(例如GPU驱动), 而Guest系统通过Hyper-V软件接口与硬件间接通讯, 该接口被称为XVIO, 一种类似于hv_sock或Linux世界virtio的半虚拟化接口.

Xbox one的HostOS是否仍基于其发布时的Windows 8不得而至. 但有理由相信Xbox series X|S的Host OS已经基于Windows10的代码开发, 但提供的功能和接口应该类似.  

### System OS / Shared OS / SRA OS

SRA是Shared Resources Access的缩写, SystemOS只能得到受限的CPU, GPU和RAM资源, 但是仍足以运行大多数UWP apps, 包括基岩版的Minecraft或者PPSSPP模拟器.

SystemOS还负责运行内置App, 如设置App, Edge, 商店, 以及整个shell(主界面, 西瓜键弹出的侧栏, 通知)

SRA可以运行两种模式的程序: App和Game.

Xbox one总共有8核AMD美洲豹CPU, 8GB DDR3内存外加32M ESRAM, 其中3G保留给System OS以运行UI和UWP, 5G用于游戏. Xbox one X共有12G GDDR5内存, 9G可用于游戏.

App在前台最多只能使用1G内存, 在后台最多只能使用128M内存, 共享2-4个CPU核(取决于目前有多少App和Game在运行), 最多45%的GPU, 最高只能使用DX11 Feature Level 10.

UWP Game可以使用最多5G内存, 独占4个CPU核并共享2个CPU核, 使用100%的GPU资源, 可以使用DX11或DX12, 但Game在后台会被暂停. UWP Game可以使用所有的GPU资源, 和没有X1X优化的ERA游戏一样多的RAM ,但是不能访问一些特有的功能, 比如多人合作的API或XDK的神秘优化.

更详细的指标可以参考[微软的UWP开发文档](https://docs.microsoft.com/en-us/windows/uwp/xbox-apps/system-resource-allocation)

System OS基于最新的Windows代码开发, 从Windows 8 一路跟随桌面Windows的步伐更新到Windows 10

例如, 编写本文时, Windows insider dev通道的最新版本为21370, 开发代号co_release钴, stable版本的最新版本为19041/2/3 (根据功能包而定), 开发代号vb_release钒. 而Xbox insider Alpha Skip-Ahead通道的版本号为2108.210430 ( XB_FLT_2108CO21374.1501.210430-2200 ), 即编译于2021年4月30日, 基于Windows 21374代码开发, 而Alpha通道的版本号为2105.210501(XB_FLT_2105VB19041.7750.210501-0000), 即编译于5月1日, 基于Windows 19041代码开发.

尽管版本号和代码基础相同, 但Windows自Windows8以来有两种不同的操作系统, 一种是完整版的Windows, 如Windows 10, Windows server, 另一种是"精简版"的Windows, 如Windows Phone 8, Windows 10 mobile, Windows10 Teams ( Surface Hub 所搭载的系统, 也称PPIPRO ), Hololens OS, Windows 10 IOT, Windows 10 X ( Windows core OS, wcos ), 以及Xbox HostOS, 这些系统都有共同的特点:

* 默认状态仅能运行UWP等winrt程序, 不能运行win32程序
* win32子系统被阉割, 如没有win32kfull.sys, 而是使用win32kmin.sys
* 由于win32子系统的阉割, 没有win32窗口的概念, 不使用Explorer+DWM, 而是使用基于UWP开发的shell
* 没有GDI, 只能用D2D或D3D绘图
* Windows常见的内置组件仍存在, 例如Windows服务, 注册表, cmd.exe等
* C盘不可写, 数据存储在其他盘符.

微软把Edge chromium浏览器移植到了wcos, 并且该功能已经发布在Xbox insider Skip-Ahead通道, 这也是wcos能运行的最win32的程序了.

### GameOS / Title OS / Exclusive OS / ERA OS

ERA是Exclusive Resources Access的缩写,  GameOS可以得到主要的CPU, GPU和RAM资源.

然而GameOS并非特指某一个操作系统, 每一个游戏的安装包都附带一个GameOS的xvd镜像, 以对应游戏开发时使用的XDK. 针对Xbox one开发的游戏, 其GameOS同样基于改造的Windows8/10, 仅包括了内核, HAL, 用于游戏的DirectX运行时, 因此大小很小, 仅80-90MB , 尽管如此之小, GameOS仍是一个可运行的Windows NT操作系统, 并且运行在HostOS提供的虚拟机里. 

(era.xvd的大小来自:

1. [Steam上发现的莫名其妙的打包错误](https://steamdb.info/patchnotes/4162835/)
2. 泄漏的XboxOne_NOV_SDK.rar中包含的DurangoXDK-November2014

)

Xbox 360模拟器游戏使用的GameOS是微软招安原Xbox 360模拟器开发者开发的官方Xbox 360模拟器, 直接模拟了Xbox 360使用的PowerPC CPU, 并在其中运行原版Xbox 360系统. 在Xbox one上使用Xbox 360光盘时, 仍需要从微软服务器下载模拟器和游戏文件, 光盘只用于验证正版. 也许该官方模拟器对每一个兼容游戏都有定制之处, 因此并非所有游戏都被兼容.

在Xbox游戏的管理游戏和附加内容->文件信息菜单可以看到GameOS的变种, 在Gen标签下

* Xenon: Xbox 360 模拟器游戏
* Durango: Xbox one 原版游戏
* Scorpio: Xbox One X 优化游戏
* Scarlett: Xbox Series X|S 次世代通用
* Anaconda: Series X 优化游戏
* Lockhart: Series S 优化游戏

在Console Type下

* Xbox360: 模拟器
* XboxOne: 为Xbox one开发
* XboxOneGen9Aware: 为Xbox one开发, 但为 X|S增强
* XboxGen9: X|S原生游戏

在AppModel下

* GDK: 第9世代游戏
* XDK: 第7/8世代游戏

(然而Minecraft并没有以上3个标签, 暴露了这个"游戏"是作为UWP Game运行而不是ERA运行的.)

通过GameOS这样独立于操作系统的另一个操作系统, Xbox解决了兼容性的问题, 也阻止了模拟器的开发, 因为甚至没有人可以解密era.xvd里到底有什么. HostOS的安全机制保证了在GameOS下试图读取C:\里era.xvd的数据就会导致虚拟机终止运行, 即GameOS在内存中的权限是仅执行. 当然, 另一方面, 如果era.xvd或微软的加密密钥被破解, 就意味着Xbox one的模拟器就会瞬间成熟. 后文我们可以看到, 开发Xbox one的模拟器甚至比破解Xbox one更简单.

基于虚拟机的运行方式也为Xbox series S|X上的快速恢复功能提供了基础, 快速恢复即等同于Hyper-V虚拟机的挂起, 只需要妥善的保存好CPU寄存器, RAM和DirectX状态即可.

## XVD

XVD, Xbox Virtual Drive的缩写, 是一种加密的磁盘镜像, 用于存储Xbox one的数据. `xvd`文件用于存储Xbox one系统的镜像和数据, 而一个变种`xvc`文件用于存储游戏镜像.

XVD的第一部分是0x0-0x200, 包含了对头部签名

XVD的第二部分是文件的头部, 包含了对顶层Hash表的Hash和文件的元数据(如content ID, content type, sandbox ID, product ID) , 这是一个微软特色的xml格式字符串.

XVD的第三部分是可选的另一个XVD文件, 通常是游戏镜像文件`xvc`中存储的GameOS镜像. 该部分是原封不动的XVD文件

XVD的第4部分是可选的Hash树, 一种多层的Hash数组. Hash树的一层会包括其下一层的块的Hash, 最低层就是对数据块的Hash. 该Hash算法为SHA-256.

XVD的第5部分是可选的本地持久存储(即用户数据区), 允许游戏或系统存储一些数据

XVD的第6部分是实际的数据区. 如果文件是`xvc`文件, 前3个块是XVC描述符, 存储了content ID, 加密的密钥, 所需要的更新包(如果游戏使用了差分更新), 以及文件中不同区域的偏移/长度/密钥ID.

XVD的最后一部分是文件系统数据, 未加密的数据为标准的NTFS元数据. 这部分数据使用内容实例密钥(Content Instance Key, CIK)加密(RSA算法). `xvd`文件的CIK存储在头部中, 并使用离线分发密钥(ODK)加密, 而游戏镜像`xvc`文件使用独立的文件License Key Packages包含的密钥. Xbox one零售系统使用的ODK并不为人之, Xbox 开发版系统使用的ODK则很容易得到, 但是无法用于任何零售游戏的解密. Windows10 游戏服务的CIK可以使用工具提取出来.

## 微软的小聪明

### 删除文件App

文件App是继承自Windows 10 mobile的一个app, 可以访问data分区受限的几个文件夹, 以及格式化为数据分区的USB存储设备. 

文件App有很多漏洞, 允许攻击者浏览HostOS的C:\中的数据, 例如在U盘上建立一个到C:\的符号链接, 并插入Xbox, 这样就可以直接以UWP沙盒的权限浏览C:\中的数据. 鉴于所有的xvd都被不明的密钥加密了, 这是少数可以得到Xbox上运行的未加密的代码和数据的方式.

然而微软以该功能用户较少为理由删除了文件App

### 遥测

微软会记录所有用户的操作和系统中正在运行的程序, 尤其是加入Insider或开启开发者模式后, 如果攻击者试图访问不应被访问的数据, 执行不应存在的程序, 遥测服务就会将其记录下来, 微软可以根据遥测数据ban掉该机器或修复系统中的漏洞

### Edge浏览器

在大多数情况下, Edge浏览器只能在在线模式中使用, 即登陆了Xbox Live, 因此如果系统不是最新版本, 或者没有链接到网络, Edge浏览器是无法打开的(提示检查网络连接). 唯一的例外是portal 认证. 通过浏览器进行破解是黑客的常规操作, 每一次PS4被破解都是因为Webkit的漏洞, EdgeHTML也不例外, 已经有几次因EdgeHTML的漏洞而攻破HostOS的例子了.

# 抵御物理攻击, Xbox one的故事

## Xbox的安全性目标

### 商业目标

* 阻止游戏/App盗版 ( 微软靠卖游戏赚钱 )
* 阻止Xbox live上的作弊 ( 公平游戏 )

### 技术目标

* 防止Xbox用户(拥有者)对设备的攻击, 维持对CPU执行的控制权
* 阻止游戏明文和密钥的泄漏 ( 任何人都无法解密加密数据, 无法获得未加密游戏文件 )
* 可以从任何软件bug恢复 ( 不会因软件错误导致整个防护体系失效 )

## Xbox 安全性不是 Windows 安全性

### Windows 安全

* PC拥有者与微软合作, 阻止安全问题
* 攻击者来自互联网

### Xbox 安全

* 攻击者是设备拥有者, 攻击者会攻击硬件
* 不能相信任何来自闪存, 硬盘, **内存**, 光盘的数据
* 任何主板上暴露的针脚都会被试图攻击

## 针对 Xbox 360 的破解

### Xbox 360: kamikaze hack

一种常见的盗版方法是对光驱的DSP进行攻击, 使得系统认为刻录的盗版盘是正版的.

Xbox 360的光驱DSP的固件位于一片闪存上, 和DSP封装到一起, 闪存的写使能引脚没有引出, 因此通常无法修改DSP的固件.

攻击者发布了一个教程, 将封装芯片的几个pin连线, 找到其交点, 即可定位到封装之下闪存的写使能引脚的位置, 然后钻透封装, 使用一个特制的芯片接入到钻孔, 当钻孔合适时, 写使能会通电, 这时给光驱通电, 即可写入修改的固件

![image](/image/image-20210505164712242.png)

### Xbox 360: Reset Glitch Hack (RGH) aka 脉冲自制系统

Xbox 360晚期的一种破解方法. Xbox 360的CPU芯片存在一个问题, 如果给芯片的reset引脚施加一个合适的脉冲, 时间不长也不短, 则芯片并没有完全重启, 而是进入了随机状态. 攻击者尝试了半年的时间, 最终找出了让CPU加载NAND上未签名二进制程序的脉冲信号. 而这个脉冲信号施加的时机必须恰到好处, 刚好处于CPU正在比较NAND上程序的签名是否为正确的签名时, 该Glitch发生, 使得这个memcmp跳转到错误的分支. 为了提高成功率, 降低对时序的要求, 该破解方法还对南桥进行了修改, 使得主CPU的频率下降, 并且修改南桥的程序, 使得Glitch成功之前不停的重启CPU.

![image](/image/image-20210505164642192.png)

## Xbox one的硬件安全性假设

1. 只有CPU是可信的 ( 对28nm的硅晶片进行修改在成本上是不可行的 ), 所有其他的硬件都不可信.

   * 拦截并修改主版总线上的数据是简单的
   * Soc任何外部的pin都有可能被按照错误的方法操纵. JTAG 调试端口, 时钟, 电压, 重置引脚是重点对象
   * 内存总线可以被拦截并修改
   * PCIE, SATA, USB数据可以轻松的被修改
2. 当破解的成本在10个游戏的价格($600)之上时, 破解无法在经济上带来收益.
3. 当前的PC硬件设备无法抵御硬件攻击
   * 内存没有加密, 没有完整性保护
   * CPU和南桥之间的总线没有得到保护
     * Intel的安全启动功能主要实现在南桥上
     * 在启动时, CPU通过不安全的总线告诉南桥它需要安全启动. 当没有受到安全启动要求时, 南桥不会安全启动
   * TPM芯片通过不安全的LPC/SPI总线连接到CPU. 任何和TPM的通讯都可以被伪造.
   * 一些工业标准的功能是insecure by design: PCIE地址翻译服务
4. 当前的软件没有针对物理攻击做出防御
   * 软件假定存储在HHD上的数据在写入和读取之间是相同的
   * 软件假定没有人可以写入Flash, 只要软件阻止该写入
5. 结论: 需要自制芯片

## Xbox Soc

![image-20210505164753707](/image/image-20210505164753707.png)

### 安全处理器(Security Complex)

* CPU ( 据称为ARM架构 )

* 加密引擎& 寄存器

* 随机数生成器
* RAM
* ROM
* OTP FUSE Bank 一次性熔断器( 用于阻止系统降级 )

* Side Channel Monitors: 监控芯片状态: 电压, 频率, Glitch

### 南桥

使用IOMMU连接到Soc, 阻止使用DMA读取关键内存数据

### 流加密引擎(Streaming Crypto Engine)

实时高性能AES解密+sha Hash

安全处理器和流加密引擎使用特殊的数据通路交换密钥, 不通过通用互联总线

硬盘上的游戏在此处解密, 且解密密钥无法被任何软件/固件访问到

### 内存

控制器实时加密/解密数据并进行完整性校验

## 安全处理器

### 目标

* 安全启动, 确保CPU启动进可信代码
* 加载密钥到流加密引擎, 解密操作系统, 游戏, 视频
* 执行DRM策略, 确保游戏和内容得到授权
* 和Xbox live进行证书握手
* 管理主机是否具有开发机(devkit)功能
* 执行熔断机制, 阻止软件降级
* 管理整个Soc的电源和频率
* 监控频率, 电压和温度, 在需要时重置芯片

### 设计理念

* 深层次防御. 需要攻击者攻破多层防线
* 保持所有游戏加密, 只有CPU die中存在原文
* 只有购买游戏后, 解密密钥才会出现在主机上. ( 安装包和授权分离 )
* 保持对CPU的控制, 只能执行微软签名的代码
* 恶意软件无法在冷重启之间持久存在
* 开发机密钥和零售密钥分离
* 可以从任意层的软件错误恢复
* 安全启动是所有安全的和谐, 必须足够强

## 多VM架构

![image](/image/image-20210505171836435.png)

前文已介绍.

## XVD

XVD是具有完整性保护和加密的VHD镜像

当读取操作发生时, HostOS的XVD驱动使用流加密引擎解密并Hash检查XVD

![image](/image/image-20210505173007380.png)

所有硬盘都不可信. 如果向硬盘写入某一个程序, 再次读取时, 并不一定是写入的内容.因此使用XVD进行完整性校验.

### 解密XVD的密钥

* 所有游戏安装包都加密并以XVD分发
* 当某一个块被读取到内存时, HostOS实时解密该块
* 解密XVD的密钥被打包为证书密钥包"License Key Packages"
* 证书密钥包只能被安全处理器使用加密引擎寄存器中的密钥打开, 且仅当该软件的版本和服务器期望的版本一致时
* 在完成解密和策略检查后, 安全处理器将最终的XVD解密密钥加载进流加密引擎, 允许HostOS来解密XVD
* CPU/安全处理器/流加密引擎无法得到XVD解密密钥是是什么, 即使存在Firmware bug/漏洞, 也无法泄漏该密钥
* 安全处理器的一些策略:
  * 授权包含授予的SocID, SocID是Soc制造时产生的128位的ID. 在线授权仅会给申请该授权的机器发放证书密钥包, 因此其他机器无法使用该证书密钥包解密XVD
  * 证书过期的时间. 一些Xbox one游戏在离线一段时间后要求必须上线才能继续游玩
  * 安全处理器, hypervisor, OS的确切版本号
  * 该授权是否可以在离线时使用

## 光盘游戏

* Xbox 360的早期盗版由于在光盘游戏上的弱点
* 问题:
  * 光驱如何测试一个光盘是正版的而不是伪造的. ( 阻止复制光盘 )
  * CPU如何相信在SATA总线另一头的光驱. ( 克隆的光驱 )
  * 如何阻止光驱被修改, 并向CPU撒谎. ( 修改后的光驱 )
* Xbox one的解决
  * 在生产时, 给安全处理器和光驱DSP之间创建共享密钥
  * 安全处理器使用该密钥验证光驱
  * 光驱的DSP芯片是微软和联发科特制的, 包括了安全启动的功能, 且安全的保存共享密钥
  * 当光驱损害时, 无法通过更换光驱来维修. 只有制造厂可以创建该共享密钥

## hypervisor强制代码完整性检查(Hypervisor enforce code integrity)

* 无法依赖数百万行代码的内核的正确性来强制执行代码签名
* 必须将该强制措施放在更底层, 且代码更少, 很有可能没有bug的地方: Hypervisor
* Hypervisor执行如下策略:

  * Hypervisor配置CPU的二级MMU来控制执行权限
  * 所有可执行页必须由微软签名
  * 页面不能是可执行且可写
  * Hypervisor假定系统有风险, 不可信
  * 系统无法添加可执行代码
* HVCI是Windows Hyper-V的功能, 起源于Xbox 360, [Windows10 也支持一个更通用的HVCI功能](https://docs.microsoft.com/en-us/windows/security/threat-protection/device-guard/enable-virtualization-based-protection-of-code-integrity)

## 操作系统状态隔离和完整性检查

* Xbox系统分为只读分区和可读写分区
* 只读分区来自XVD, 具有完整性检查
* 可写分区只有注册表和设置, 没有代码
* 系统更新就是把旧的XVD换成新的XVD
* 因为系统完整性检查是在启动时进行的, 即使攻击者找到一个bug来执行恶意代码, 一个冷重启就会将系统重置
* 恶意代码无法在冷重启之间持久存在, 不存在rootkit

## 安全处理器 密钥寄存器

### 功能

* 记录所启动的系统的所有bootloader到硬件PCR(Platform Configuration Register, 平台配置寄存器, TPM 2.0术语) 作为Attestation(证据), 以供Xbox Live检查. Attestation的产生具体该过程见下文.
* 保存全局密钥 ( 所有Xbox都具有相同的密钥 )
* 保存机器密钥 ( 每台机器一个单独的密钥 )
* 保存游戏解密密钥

### 安全目标:

* 即使攻击者可以在安全处理器上运行任意代码, 其仍不能得到有价值的密钥(即解密xvd的密钥), 也不能销毁其破解操作产生的Attestation. 攻击者仅能获得当前版本的软件使用的密钥, 更新软件即可使该密钥失去价值.
* 微软可以修复bug, 发布软件更新, 包括新的密钥来解密游戏的证书密钥包. 没有更新的系统无法获得新的证书.

## 全局密钥树

![image](/image/image-20210505200016480.png)

* 每一阶段的密钥负责解密和验证下一阶段的代码
* 无法读取父节点或兄弟节点的密钥, 因为该密钥已经从内存中删除(父节点)或不存在(兄弟节点)
* Xbox共有10级以上的加载流程
* 任何一个阶段的改变都会导致下一阶段的解密失败

## 安全启动

* 必须将下一阶段完整的放入Soc, 防止TOCTOU bug. TOCTOU即Time-of-check to time-of-use, 当检查和使用分离时, 如果在检查后使用前所用对象发生了变化, 则检查实际上是无效的. 例如假如下一阶段的Bootloader位于NAND上, 当Soc检查签名和完整性后, 重新从硬盘上读取Bootloader, 则攻击者可能更换NAND中的内容.
* 每一阶段的摘要都被记录进PCR寄存器用于Attestation
* 每一阶段都使用全局密钥树中的没密钥加密
* 每一阶段的摘要都存入全局密钥树
* 成功解密和验证下一阶段依赖于前一阶段存入全局密钥树的摘要是正确的
* 当系统启动到阶段N, 解密更早阶段的密钥将被删除
* 如果所有分支都没有错误, 会启动到正确的系统
* 在验证该阶段的代码是否正确之前, 这一阶段的摘要都已经被记录进PCR寄存器和全局密钥树. ( 试图修改某一阶段会污染PCR和全局密钥树, 导致无法解密所有的后续阶段 )
* 会同时校验密文和明文的摘要
* 启动后, 安全处理器会封锁所有安全处理器的固件代码

## Attestation (证据)

* 证据是用来安全的测量在本次启动过程中所使用的所有软件的摘要, 并将其证明给Xbox Live等服务.
* 每一启动阶段都计算下一个阶段的SHA摘要, 并将其存入PCR寄存器
* 安全处理器的Boot ROM会记录第一阶段安全处理器固件的摘要
* 一个损害的"阶段"无法回退并对其摘要撒谎
* 在加密寄存器中有一个每一台主机单独的私钥, 用于签署PCR. 这一私钥不能用于其他目的.
* 对应的公钥存储在每一台主机单独的证书中, 在Xbox制造过程中签署
* 唯一无法恢复的软件错误就是Boot ROM中存在的bug ( 说的就是你, checkm8 )

注: PCR存储的过程. 假设存储的是SHA-1, 其大小是20字节, 那么PCR寄存器的大小就是20字节, 向PCR存储数据的过程实际上被称为extended(扩展), 其值按照该公式计算: `PCR = Sha1("$PCR" + "$Value")` 即PCR新值等于老值这20字节后门跟上输入值这20字节,然后再取这40字节的SHA-1. 因为Hash被认为是不可逆的, 因此输入一个不正确的值就会直接污染本次启动流程, 使得PCR寄存器的内容几乎无法恢复到正确的值上.

参考: [基于静态可信根(SRTM)的Bitlocker的工作原理是什么？](https://zhuanlan.zhihu.com/p/33858479)

Attestation这一机制实现了如下功能, 当Xbox Live检测到一个老版本的摘要发来的请求, 就会强制要求该机器离线或更新系统, 而如果收到一个从来都不应出现的摘要, 就可以直接ban掉该主机.

## 为何Xbox的安全性如此持久

注: Xbox团队2011年开始研发安全机制, 2013年Xbox one发售, 其演讲使用的PPT在2014年完成, 并多次在微软内部演讲. 本次演讲, Platform Security Summit 2019, 是Tony chen第一次公开Xbox 背后的安全机制. 而至今(2021年), 8年过去了, 仍没有任何人攻破Xbox one的HostOS或GameOS.

* 保持关键组件越简单越好(如 Boot ROM, Hypervisor)
* 使所有启动代码加密使得攻击者很难搞清楚启动时发生了什么, 如何去攻击代码. ( 本文也只是介绍了安全部分, 至于十几个阶段到底干了什么, Xbox 安全团队之外的人很难得知 )
* 启动流程被设计出来可以抵抗Glitch
* 使用一个独立的安全处理器, 具有自己的ROM, SRAM, RNG, Fuse, Crypyo keys registers和加速器
* 攻击者知道微软可以更新所有的软件并使他们的工作无效, 这降低了(以盈利为目的的 )对Xbox one的攻击欲望
* 仔细的review, 威胁模型分析, 渗透测试来避免愚蠢的错误
* 可以抵抗硬件攻击的硬件, 以及可以抵抗硬件攻击的软件设计

## Q&A

Q: 为什么不能购买一个Microsoft Surface pro with Xbox inside

A: 1. 当前硬件的安全启动不可靠 2. 用户期望PC可以运行win32, 但是Xbox不需要运行

Q: 最初Xbox one被宣称为always online & 没有光驱, 之后有什么重大的更改来支持离线 & 光盘游戏

A: 在发售前6个月, Xbox开发团队被告知需要支持光盘游戏. 通过和MTK的合作, DSP包括了相同逻辑设计的安全系统, 以及要求光驱和Soc进行唯一的密钥匹配.

Q: Root key是每一台主机一个还是通用的

A: Global key是用于安全启动的key, 是所有Soc相同的, per-console key用于未说明的其他目的. Global key也可以用于解密光盘游戏, 因此从未更新过的系统也可以玩更新的光盘游戏.

Q: DRAM是否使用相同的key加密

A: DRAM有自己的key来加密

Q: Xbox update如此巨大是否是因为xvd的使用, 是否支持delta更新

A: 系统的xvd并不使用delta更新 ( 存储游戏的xvc支持delta patch )

Q: 人们接受了要玩新游戏就要更新系统的策略, 且破解只能作用于本地, 这听起来像蓝光光盘的策略

A: 的确像蓝光的密钥一样, 如果有人成功破解了Xbox one, 那么就像蓝光播放机的策略一样, 在新的游戏上使用新的key, 然而旧的游戏就只能被破解了. 虽然这并没有发生.

Q: 微软和AMD合作, 但是AMD在固件安全方面的名声不好. 听起来安全处理器和AMD PSP(Platform Security Processor)一样, 那么是AMD编写了安全处理器的固件还是微软?

A: 微软编写了相关的固件, 且开发了相关的硬件. AMD PSP的推出晚于Xbox one

Q: 如何防止开挂

A: Xbox的安全机制阻止了运行未签名代码, 基本阻止了作弊的可能性.

Q: Root key是否可以更新

A: 不能, Root key是出场时写入Fuse的
