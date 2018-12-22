+++
title = "在FydeOS上安装ArchLinux"
date = 2018-12-22
[taxonomies]
categories = ["Linux"]
tags = ["Linux","ChromiumOS"]
+++

# 前言

FydeOS是一个国产的ChromiumOS发行版,是目前唯一像ChromeOS一样支持安装Android应用的ChromiumOS发行版.FydeOS像ChromeOS一样也支持运行Linux程序,但是默认的发行版是Debian,包又老又少,因此决定换Archlinux.
<!-- more -->

# Crostini结构

ChromiumOS运行Linux的项目叫Crostini.[ChromiumOS的文档](https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md#Glossary)详细的介绍了其架构.

    ChromiumOS
        Chrome浏览器

        Android运行环境
            Android应用

        crosvm虚拟机
            Termina
                Linux容器
                    penguin
                    archlinux
                    其他Linux容器实例

            其他虚拟机实例

这里(机器)翻译/修改一下官方文档,以便理解其中的法文命名的专业名词.

Crostini是使Linux应用程序支持易于使用并与Chrome OS完美集成的总称。它侧重于为您提供一个容器的终端，可以轻松访问以开发人员为中心的工具。与此对比,Crouton提供了非官方的Linux支持,不是很安全,也不是很方便,但特权更大。

crosh是Chrome OS上使用前端技术构建的一个终端模拟器，使用CTRL-ALT-T可以打开它。

终端应用是该环境的第一个入口。它基本上只是crosh的快捷方式。

crosvm是一个虚拟机监视器，负责管理KVM，guest VM以及进行低级（基于Linux virtio驱动）的通信。

Termina是一个虚拟机映像，带有精简的Chrome OS linux内核和用户态工具。它的唯一目标是尽快启动并开始运行容器。许多程序/工具都是自定义的。其名字是“终端terminal”缺少一个字母，但并非有意如此。

Concierge(礼宾部门)是一个在Chrome OS中运行的守护进程，它处理VM和容器的生命周期管理，并使用gRPC over vsock与Maitred进行通信。

Maitred(侍者总管)是VM内部的init和服务/容器管理器，负责与Concierge（在VM外部运行）进行通信。Concierge发出请求，Maitred负责执行这些请求。

Tremplin(法国著名酒店)是一个在VM中运行的守护进程，为LXD提供gRPC的包装。这包括一些基本功能，如创建和启动容器，还提供其他Crostini特定的集成，例如设置容器的主用户，以及在guest虚拟机中设置匹配Chrome OS版本的apt存储库。

Cicerone(侍酒师认证)是一个在Chrome OS中运行的守护程序，它在容器开始运行后直接处理与VM和容器的所有通信。具体来说，它与Tremplin（在VM内部运行）和Garcon（在VM内部的容器中运行）进行通信。

Garcon(侍者)在容器内部运行，并提供与Cicerone/Chrome的集成，以实现更方便/自然的行为。例如，如果容器想要打开URL，Garcon会负责管理该请求。

9P协议是[Plan 9（贝尔实验室九号计划）](https://zh.wikipedia.org/zh-cn/Plan9)的遗产,一种优秀的网络协议,能远程的访问文件,就像访问本地文件一样。贝尔实验室解散后,大多数科学家都来到了Google,并将9P移植到Linux.

Seneschal(总管)是一个在Chrome OS中运行的守护进程，用于处理9P服务器的生命周期管理。当Concierge启动VM时，它会向Seneschal发送一条消息，以便为该VM启动9s实例。然后，在配置VM时，Concierge会向Maitred发送一条消息，指示它连接到9s实例并将其挂载到VM中。

9s是9P文件系统协议的服务器。每个VM都有一个9s的实例，它使VM可以访问存储在VM外部的用户数据。其中包括“下载”文件夹，Google云端硬盘和可移动媒体等内容。每个9s实例的生命周期由Seneschal管理。每个9s实例启动时无法访问任何文件。通过向Seneschal发送消息来授予对特定路径的访问权限，这使得所请求的路径可用于指定的9s实例。共享路径的请求只能由某些用户操作触发。

Sommelier(侍酒师)是Wayland代理合成器，在容器内运行。Sommelier可以在容器内的Wayland应用程序和Chrome之间无缝转发内容，输入事件，剪贴板数据等。Chrome OS不支持X协议，因此，Sommelier还负责启动XWayland（以rootless模式），充当客户端的X窗口管理器，并将容器内的X协议转换为Chrome OS的Wayland协议。

penguin(企鹅)是默认的容器。

# 安装Archlinux容器

使用CTRL-ALT-T可以打开打开`crosh`.不要输入`shell`进入`bash`.输入以下命令以测试你来到了正确的地方,并查看`vmc`,虚拟机控制命令的用法.

```output
crosh> vmc --help
vmc [ start <name> | stop <name> | destroy <name> | export <vm name> <file name> [removable storage name] | list | share <vm name> <path> ] Manage a VM.
```

启动`Termina`

```output
crosh> vmc start termina
(termina) chronos@localhost ~ $
```

查看Linux容器命令`lxc`的用法.

```output
(termina) chronos@localhost ~ $ lxc --help
....
```

列出已经安装的容器.

```output
(termina) chronos@localhost ~ $ lxc list
+---------+---------+-----------------------+------+------------+-----------+
| NAME    | STATE   | IPV4                  | IPV6 | TYPE       | SNAPSHOTS |
+---------+---------+-----------------------+------+------------+-----------+
| penguin | RUNNING | 100.115.92.202 (eth0) |      | PERSISTENT | 0         |
+---------+---------+-----------------------+------+------------+-----------+
```

使用`run_container.sh`命令可以下载并安装`Archlinux`容器.
由于FydeOS相对于ChromiumOS对此命令进行了修改,编辑这个sh脚本撤销更改.

```output
(termina) chronos@localhost ~ $ cp /usr/bin/run_container.sh /tmp
(termina) chronos@localhost ~ $ cd /tmp
(termina) chronos@localhost ~ $ vim run_container.sh
```

找到以下片段

```bash
# lxc init "google:${FLAGS_lxd_image}" "${FLAGS_container_name}" || \
#   die "Unable to create container from image '${FLAGS_lxd_image}'"
local container_root="/usr/share/intergrade_container"
local lxd_info="${container_root}/lxd.tar.xz"
local root_file="${container_root}/rootfs.squashfs"
local container_alias="intergrade_fydemina"
lxc image import $lxd_info $root_file --alias $container_alias || \
    die "Unable to import image from $root_file"
lxc init $container_alias ${FLAGS_container_name} || \
    die "Unable to create container from image $container_alias"
```

修改为

```bash
lxc init "google:${FLAGS_lxd_image}" "${FLAGS_container_name}" || \
    die "Unable to create container from image '${FLAGS_lxd_image}'"
```

运行以下命令.请确保用户名是设置应用里显示的用户名.你可以自行选择`container_name`指定的镜像名,`lxd_image`指定的linux镜像id,或者`lxd_remote`指定的镜像源.

```output
(termina) chronos@localhost /tmp $ bash ./run_container.sh --container_name arch --user 你的用户名 --lxd_image archlinux/current --lxd_remote https://mirrors.tuna.tsinghua.edu.cn/lxc-images/
```

确保下载成功并且创建用户成功(忽略那几个用户组的错误).启动镜像的shell

```output
(termina) chronos@localhost /tmp $ lxc exec arch -- bash
[root@arch ~]#
```

在容器中执行以下命令

```bash
#设置密码.千万不要给root设置密码,否则ChromiumOS集成服务将无法运行
passwd 你的用户名

#把用户加入wheel组
usermod -aG wheel 你的用户名

#设置源,把tuna ustc等中国的镜像源剪切粘贴到前面.vim中dd剪切整行,p粘贴,/搜索
vi /etc/pacman.d/mirrorlist

#设置archlinuxcn源等
vi /etc/pacman.conf

粘贴以下两行

[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch

#安装依赖
pacman -Syu base-devel git gtk3 openssh xdg-utils xkeyboard-config archlinuxcn-keyring

#启用sudo无密码
visudo

删除以下行前的注释#

%wheel   ALL=(ALL:ALL) NOPASSWD: ALL

#退出
exit
```

使用另一种方式**登录**到你创建的用户(之前的执行bash的方法不是登录,无法加载服务).

运行`lxc console arch`

```output
(termina) chronos@localhost /tmp $ lxc console arch
To detach from the console, press: <ctrl>+a q
你的用户名
Password:
[你的用户名@arch ~]$
```

登录成功后安装aur上的`cros-container-guest-tools-git`.由于需要从`chromium.googlesource.com`下载文件,因此自行解决网络故障.注意,Android或者Chromium OS里的代理设置不会应用到虚拟机.

```bash
git clone https://aur.archlinux.org/cros-container-guest-tools-git.git
cd cros-container-guest-tools-git
makepkg -i
#解决.config不存在导致没有设置默认浏览器的问题
mkdir ~/.config
xdg-settings set default-web-browser garcon_host_browser.desktop
```

解决Archlinux里`xkeyboard-config`太新`Sommelier`不支持两个键码的问题
打开`/usr/share/X11/xkb/keycodes/evdev`,注释或者删除`<i372>` 和 `<i374>`开头的两行.

`cros-container-guest-tools-git`应用于ChromiumOS的最新版本,FydeOS暂时没有更新到R71,不支持`x-auth`功能,因此需要修改文件.

打开`/usr/lib/systemd/user/sommelier-x@.service`,把

```bash
ExecStart=/opt/google/cros-containers/bin/sommelier \
              -X \
              --x-display=%i \
              --sd-notify="READY=1" \
              --no-exit-with-child \
              --x-auth=${HOME}/.Xauthority \
              /bin/sh -c \
                  "systemctl --user set-environment ${DISPLAY_VAR}=$${DISPLAY}; \
                   systemctl --user set-environment ${XCURSOR_SIZE_VAR}=$${XCURSOR_SIZE}; \
                   systemctl --user import-environment SOMMELIER_VERSION; \
                   touch ${HOME}/.Xauthority; \
                   xauth -f ${HOME}/.Xauthority add ${HOST}:%i . $(xxd -l 16 -p /dev/urandom); \
                   . /etc/sommelierrc"
```

改为

```bash
ExecStart=/opt/google/cros-containers/bin/sommelier \
              -X \
              --x-display=%i \
              --sd-notify="READY=1" \
              --no-exit-with-child \
              /bin/sh -c \
                  "systemctl --user set-environment ${DISPLAY_VAR}=$${DISPLAY}; \
                   systemctl --user set-environment ${XCURSOR_SIZE_VAR}=$${XCURSOR_SIZE}; \
                   systemctl --user import-environment SOMMELIER_VERSION; \
                   . /etc/sommelierrc"
```

启用`cros-container-guest-tools-git`附带的所有systemd服务

```bash
sudo systemctl enable cros-sftp
systemctl --user enable sommelier@0.service
systemctl --user enable sommelier@1.service
systemctl --user enable sommelier-x@0.service
systemctl --user enable sommelier-x@1.service
systemctl --user enable cros-garcon.service
```

由于一些限制,目前`Crostini`的Chromium OS集成仅名为`penguin`的容器可以启用,因此需要重命名容器.不要删除自带的Debian容器.

```bash
[你的用户名@arch ~]$ exit
按下ctrl-A Q
(termina) chronos@localhost /tmp $ lxc stop arch
(termina) chronos@localhost /tmp $ lxc stop penguin
(termina) chronos@localhost /tmp $ lxc rename penguin debian
(termina) chronos@localhost /tmp $ lxc rename arch penguin
```

随后重启系统.

打开终端应用,等待几秒,archlinux就启动了,随后做一些本地化

首先设置中文语言

创建`$HOME/.config/locale.conf`

```bash
cat << EOF > $HOME/.config/locale.conf
LANG="zh_CN.UTF-8"
LANGUAGE="zh_CN.UTF-8"
LC_CTYPE="zh_CN.UTF-8"
LC_NUMERIC="zh_CN.UTF-8"
LC_TIME="zh_CN.UTF-8"
LC_COLLATE="zh_CN.UTF-8"
LC_MONETARY="zh_CN.UTF-8"
LC_MESSAGES="zh_CN.UTF-8"
LC_PAPER="zh_CN.UTF-8"
LC_NAME="zh_CN.UTF-8"
LC_ADDRESS="zh_CN.UTF-8"
LC_TELEPHONE="zh_CN.UTF-8"
LC_MEASUREMENT="zh_CN.UTF-8"
LC_IDENTIFICATION="zh_CN.UTF-8"
EOF
```

安装字体,输入法

```bash
sudo pacman -S fcitx-im fcitx-configtool fcitx-sogoupinyin wqy-microhei
```

打开`/usr/lib/systemd/user/cros-garcon.service.d/cros-garcon-override.conf`

插入

```bash
Environment="GTK_IM_MODULE=fcitx"
Environment="QT_IM_MODULE=fcitx"
Environment="XMODIFIERS=@im=fcitx"
```

执行

```bash
echo /usr/bin/fcitx-autostart > $HOME/.sommelierrc
```

重启系统.