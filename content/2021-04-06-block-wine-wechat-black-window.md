+++
title = "屏蔽微信在Wine中运行时产生的水印/黑块（Linux Edition）"
date = 2021-04-06
[taxonomies]
categories = ["Linux"]
tags = ["Wine","Linux"]
+++

众所周知，微信并没有官方开发的Linux客户端，要想在Linux上使用微信，就产生了很多种变通方法：

- 网页版
- Wine+微信 PC版
- 虚拟机+微信 PC版
- Android虚拟机/容器+微信平板模式
- 在其他设备上使用，并使用RDP/scrcpy等软件远程控制

在这些解决方案中，微信 PC版在操作逻辑和功能上较其他方案更强，但是由于微信 win32程序会创建一个独立的半透明的窗口作为其窗口边框和UI控件的阴影特效，而在Linux下，Wine会将这个透明的窗口错误的绘制，产生一些奇怪的bug， 例如这个透明边框总是悬浮在所有窗口的最上面，即使已经切换到其他的工作区，或者由于Xwayland的背景是黑色的，这个半透明的窗口会继承黑色的背景，导致其效果不再是一个阴影框，而是一个覆盖整个窗口的黑色块。

<!-- more -->

解决这个问题有3种方法

- 使用X api隐藏半透明窗口
- 使用Win32 api隐藏半透明窗口
- 在Wine的源码里动手脚，不创建该窗口

## 修改Wine

第3种方法最直接，该方案的补丁如下：

```patch
--- a/dlls/user32/win.c  2020-04-09 22:20:30.459968806 +0800
+++ b/dlls/user32/win.c  2020-04-09 21:54:09.588643751 +0800
@@ -1795,6 +1795,17 @@
     cs.lpszClass      = className;
     cs.dwExStyle      = exStyle;
 
+    if (exStyle == 0x080800a0) // WeChat/WxWork shadow hwnd
+    {
+        FIXME("hack %x\n", cs.dwExStyle);
+        return NULL;
+    }
+    if (exStyle == 0x000800a0) // Netease Cloudmusic shadow wnd
+    {
+        FIXME("hack %x\n", cs.dwExStyle);
+        return NULL;
+    }
+
     return wow_handlers.creat[e_window( &cs, className, instance, TRUE );
 }
```

exStyle是win32 api中每一个窗口都具有的一个属性:[扩展窗口样式](https://docs.microsoft.com/en-us/windows/win32/winmsg/extended-window-styles)，这个有bug的窗口打开了工具栏窗口`WS_EX_TOOLWINDOW`,透明窗口`WS_EX_TRANSPARENT`,层叠式窗口`WS_EX_LAYERED`这3个标志位. 因此在创建窗口的函数处检测是否使用这个标志位即可避免该窗口的创建.

该方案目前收录于`archlinuxcn`仓库, 但是我并不使用archlinux. 由于需要编译Wine，而我并没有一个可用的glibc multilib编译环境，因此我放弃了该方案。

## Win32 API

第二种方案,使用Win32 api来隐藏窗口, 这也是我最早见到的方案, 来源于https://blog.kangkang.org/index.php/archives/397

```c++
#include "windows.h"
LPCTSTR windowClassName = L"popupshadow";

int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, PWSTR pCmdLine, int nCmdShow) {
  for (;;) {
    for (HWND h = FindWindowEx(NULL, NULL, windowClassName, NULL);
        h = FindWindowEx(NULL, h, windowClassName, NULL);
        h != NULL) {
      ShowWindow(h, SW_HIDE);
    }
    Sleep(3000);
  }
  return 0;
}
```

编译方法:

```cmd
i686-w64-mingw32-g++ -municode -m32 -static-libgcc -o wechatkill wechatkill.cpp
```

该方案使用`FindWindowEx`找到class name为`popupshadow`的窗体,然后hide掉这个窗体. 不过不知道是不是微信更新了,我编译该程序并运行后没有什么效果.

## X api

第一种同样是隐藏窗体,但是在X的层面进行的操作. 然而网上使用的方法大多是错误的, 要么是漏杀, 要么是错杀. 我写了一个正确的程序, 可以正确的隐藏阴影框, 同时不影响各个菜单.

```C
#include <stdbool.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <xcb/xcb.h>
#include <xcb/xcb_aux.h>
#include <xcb/xcb_event.h>
#include <xcb/xcb_icccm.h>
#include <xcb/xproto.h>

static xcb_connection_t *conn;
static xcb_screen_t *scr;
static xcb_atom_t atom;

void init_atom() {
  // Get `_NET_WM_NAME` atom
  const char *atom_name = "_NET_WM_NAME";
  xcb_intern_atom_cookie_t atom_cookie =
      xcb_intern_atom(conn, true, strlen(atom_name), atom_name);
  xcb_intern_atom_reply_t *atom_reply =
      xcb_intern_atom_reply(conn, atom_cookie, NULL);
  if (!atom_reply) {
    fprintf(stderr, "_NET_WM_NAME atom not found, WTF?\n");
    return;
  }
  atom = atom_reply->atom;
  free(atom_reply);
}

void handle_wechat(xcb_window_t window) {
  // Get value of `_NET_WM_NAME`
  xcb_icccm_get_text_property_reply_t prop;
  if (!xcb_icccm_get_text_property_reply(
          conn, xcb_icccm_get_text_property(conn, window, atom), &prop, NULL)) {
    fprintf(stderr, "Can't get _NET_WM_NAME property\n");
    return;
  }
  if (prop.name_len) {
    printf("Normal window with name: %.*s\n", prop.name_len, prop.name);
    return;
  }
  // If `_NET_WM_NAME` is empty, check if this windows accept input
  xcb_icccm_wm_hints_t hints;
  if (!xcb_icccm_get_wm_hints_reply(conn, xcb_icccm_get_wm_hints(conn, window),
                                    &hints, NULL)) {
    fprintf(stderr, "Can't get WM_HINTS property\n");
    return;
  }
  if ((hints.flags & XCB_ICCCM_WM_HINT_INPUT) && hints.input) {
    printf("Normal dialog without name\n");
    return;
  }
  printf("Black shadow window, unmap it!\n");
  xcb_unmap_window(conn, window);
  return;
}

bool is_wechat(xcb_window_t window) {
  xcb_get_property_cookie_t cookie = xcb_get_property(
      conn, 0, window, XCB_ATOM_WM_CLASS, XCB_ATOM_STRING, 0, 32);
  xcb_get_property_reply_t *reply = xcb_get_property_reply(conn, cookie, NULL);
  if (!reply) {
    return false;
  }
  int len = xcb_get_property_value_length(reply);
  if (!len) {
    free(reply);
    return false;
  }
  char *property = (char *)xcb_get_property_value(reply);
  printf("0x%0x8: WM_CLASS= %.*s\n", window, len, property);
  bool result = false;
  if (!strcmp(property, "wechat.exe")) {
    printf("Wechat found!\n");
    result = true;
  }
  free(reply);
  return result;
}

int main(int argc, char **argv) {
  conn = xcb_connect(NULL, NULL);
  if (xcb_connection_has_error(conn)) {
    fprintf(stderr, "Failed to connect to the X server\n");
    exit(1);
  }
  scr = xcb_setup_roots_iterator(xcb_get_setup(conn)).data;
  if (!scr) {
    fprintf(stderr, "Failed to get X screen\n");
    exit(2);
  }

  uint32_t val[] = {XCB_EVENT_MASK_SUBSTRUCTURE_NOTIFY};
  xcb_change_window_attributes(conn, scr->root, XCB_CW_EVENT_MASK, val);

  init_atom();

  while (1) {
    xcb_aux_sync(conn);
    xcb_generic_event_t *e = xcb_wait_for_event(conn);
    if (XCB_EVENT_RESPONSE_TYPE(e) == XCB_MAP_NOTIFY) {
      xcb_map_notify_event_t *map = (xcb_map_notify_event_t *)e;
      if (is_wechat(map->window)) {
        handle_wechat(map->window);
      }
    }
    free(e);
  }

  if (conn) {
    xcb_disconnect(conn);
  }
  return 0;
}
```

编译方法:

```shell
gcc xwechathide.c -lxcb -lxcb-util -lxcb-icccm -o xwechathide
```

该程序的原理: 在根窗口上监听子窗口的事件, 当收到`XCB_MAP_NOTIFY`事件时, 即一个X窗口已经map(显示)到桌面上了, 这时检查其`WM_CLASS`属性是否为`wechat.exe`. 如果是则检查`_NET_WM_NAME`(即窗口标题)是否为空, 那些阴影窗口的标题为空, 而具有功能的窗口不为空.如果标题为空, 再检查`WM_HINTS`属性中输入字段是否为允许输入, 菜单、小程序列表等对话框没有窗口标题，但允许输入，因此不应隐藏，而阴影窗口不允许输入,因此应该隐藏.

该程序正确的保留了菜单、小程序列表等对话框, 且实时响应, 无需sleep式的轮询, 仅当新的X事件时会运行, 因此效率更高, 几乎不占用CPU (微信的wechat.exe运行一个CPU小时而该程序仅运行不到1个CPU秒), 没有闪烁的情况.

( 测试环境: Flatpak FreeDesktop 20.08 runtime, wine-tkg 6.4, 禁用dxvk, 模拟Windows7, 32位prefix, 使用MS的riched20.dll和ole32.dll, 官网下载的微信, fakechinese安装的思源cjk字体 )

## Web

此部分与主题无关.

其实我的帐号并不能直接登陆进微信Web版, 但之前UOS发布了一个使用古董electron打包的**MIT**授权的微信客户端, 一眼就能看出来和网页版一模一样, 但是却可以让我的账户登陆上本来不能登陆的Web版. 不管统信他们是不是写错授权了, 但既然这唯一发布的一版是MIT授权的, 我就拆包看了看Javascript代码, 并得出了其中的原理.

首选, 必须使用该链接登陆:

```url
https://wx.qq.com/?&lang=zh_CN&target=t
```

其次, 使用chromium系列的浏览器, 安装[Header Editor](https://he.firefoxcn.net/zh-CN/guide.html)扩展.(由于Firefox的bug, 只能使用chromium浏览器).

然后设置以下3个Header头修改

```txt
匹配类型: 网址前缀
匹配规则: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage
执行类型: 常规
头名称: extspam
头内容: Gv8GCPkGEvkGCggwMDAwMDAwMRAGGuAGomtlq/mOhQXWue473LKvUjEhatKQ338GLRNavgAhb2ET3z+3ejCXaYlrG7AfJ+QBphMSYvPhxQVyhwVvQarv9muy8HSoAAuQtV1DjpOIjQwcoa/wOCicjtUV8yp2W6vTrgea200WY7+vg95AT43/qDQ0WVSupawUWCKWuXNdvxsIPp/Y4d+YHkCvOV21/uea4WqILbfOZhd4OZsX6MEHcB322DXZZ0m89mSf0drCb+8mKEMLCGbXBCy5ZFD/YR0NTsaYFQTIDkF9eBCsi56MVNWYcp0+a51Tg2Cg2fNPFPtRZ7D5TlKftMTmKy1DAVVKWeXJqXjUyATQA+ZXRW3vrdP1pc6OMWxdhgR+71VDa++pfLO/kRJ5hUgpwR8/JzfOXEqGsi866CGy/J+wOXQUtXKew+RYXYFLpspq1Tbi61yYowsBUnSl8c8JIekAdqHTSQPX5qHij58QakUblQhm2czTcHI2+j3avjsRV9aqFHmJfSRcgwYBEEVxxyzK3/cJtTzWsBpW6N6VDdby0YExETdIRM5zaElXjg/AZqHxZCuhYJS3inR5aG5ypURZVqGWYIc9AJHJCu7PKrajIX1j4KlXgROyCxAov9uPLkd/fgjoIrg5rP2pIdyM+2nNbzcnsJj+DFxwa8JiURu08ez+sX43hC7WSSQ5205k2DrEb2Tp0iHLZ9QbbzLpnZsqaufMngKzvXPwq8GrFkJ+JIUkjfe9MQaNW/CONdkqRyAZDasJO9kHpYIFcP++A+9ErWsaUXm9WsCpkiTGJBeiS3u0gqBFXAS0oa128becqhFpK5LBOSxnk2YhUP2c7AJt1sv1Ysqo+6/Vh+AI09kDjontg/P6xqKEQGLdArlppJqeaUOOhs0270am/7YWhNqS69mpcslRkvFSyCAbI26g4lppzYp4U28rmFifSplR8NI6IO0dAWr4pS5mmov4pNA6WvvBpm078JaVlXOLOZj20T8JB08yNTZcFblD54Sn78ro4ooKsaA0ynUZjvFKvlKno64dqLlnFJudmR0Qx8yFcHQy/7Qqj+/AnjdLOJ0e7rAGMvoMIlnJnn6okhOeEKfCaZpG+v54ZWFeZs7f8SitfVlfEEhCQEHLTgRiu18PVwi7gHILzlQQoiFwUhBiKrA9Tnv6IIvtsP8FKAUwAA==

匹配类型: 网址前缀
匹配规则: https://wx.qq.com/cgi-bin/mmwebwx-bin/webwxnewloginpage
执行类型: 常规
头名称: client-version
头内容: 2.0.0

匹配类型: 网址前缀
匹配规则: https://wx.qq.com/
执行类型: 常规
头名称: user-agent
头内容: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) ??/2.0.0 Chrome/59.0.3071.115 Electron/1.8.8 Safari/537.36

(此处的??实为"微信"两个汉字, 但根据规范, HTTP头应为ASCII可见字符, 因此不能存储汉字)
```

然后就能正常的登陆了.
