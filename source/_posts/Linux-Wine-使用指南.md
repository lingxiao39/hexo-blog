---
title: Linux Wine 使用指南
date: 2023-10-08 00:00:00
tags: Linux
---

## Wine 是什么

> Wine is a compatibility layer capable of running Microsoft Windows applications on Unix-like operating systems. Programs running in Wine act as native programs would, without the performance/memory penalties of an emulator.

简单来说，Wine 可以让你在 Linux 系统上无缝运行 Windows 下的应用程序，而且相比于虚拟机可以获得更好的性能。

## 安装过程

首先确保你使用的是 X11 桌面，已知 Wayland 桌面环境有兼容性问题。

输入以下命令安装 Wine 以及相关组件：

```bash
sudo pacman -S wine wine-mono wine_gecko wine-staging giflib lib32-giflib libpng lib32-libpng libldap lib32-libldap gnutls lib32-gnutls \
    mpg123 lib32-mpg123 openal lib32-openal v4l-utils lib32-v4l-utils libpulse lib32-libpulse libgpg-error \
    lib32-libgpg-error alsa-plugins lib32-alsa-plugins alsa-lib lib32-alsa-lib libjpeg-turbo lib32-libjpeg-turbo \
    sqlite lib32-sqlite libxcomposite lib32-libxcomposite libxinerama lib32-libgcrypt libgcrypt lib32-libxinerama \
    ncurses lib32-ncurses opencl-icd-loader lib32-opencl-icd-loader libxslt lib32-libxslt libva lib32-libva gtk3 \
    lib32-gtk3 gst-plugins-base-libs lib32-gst-plugins-base-libs vulkan-icd-loader lib32-vulkan-icd-loader
```

一般来说，安装完 Wine 之后直接双击 exe 程序即可运行。但是有些应用可能在 64 位 Wine 下无法运行，此时就需要手动切换到 32 位 Wine 环境：

```bash
# 设置架构为 win32，配置文件保存在 `~/.win32` 目录下
# 打开 Wine 配置界面
WINEARCH=win32 WINEPREFIX=~/.win32 winecfg
# 运行程序
WINEARCH=win32 WINEPREFIX=~/.win32 wine /path/to/program.exe
```

如果你需要运行 GalGame，则可能需要执行以下命令安装额外组件：

```bash
sudo pacman -S winetricks d3dx9 quartz devenum wmp10 gdiplus dotnet40 ffdshow cjkfonts
```

## Wine 前缀管理

一般来说，可以为每个软件新建一个 Wine 环境，使用单独的配置运行每个 exe 程序，这时候就需要进行前缀管理。

我在 `~/.zshrc` 文件中配置了默认使用的 Wine 前缀和架构如下：

```bash
export WINEARCH=win32
export WINEPREFIX=~/.win32
```

这样就不用每次执行 exe 时都要在前面加上 WINEARCH 参数了。

此外，还可以为每个 exe 文件编写运行脚本，一键运行：

```bash
#!/bin/bash

WINEARCH=win32
WINEPREFIX=~/.win32
wine /home/username/.win32/drive_c/altera/13.0sp1/quartus/bin/quartus.exe
```

## 参考资料

1. [Wine - ArchWiki](https://wiki.archlinux.org/title/Wine)
